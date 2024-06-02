# Exposing Packets to JS

Basically, hooking involves inserting a line of code into the existing code to execute the desired logic. The same applies to [core hooking](core.md). BDSX implements hooking using an API provided by the core, allowing access to actual memory through pointers in JS.

Assembly code is not special. As with writing plugins in TS, we write the logic we want, just in the assembly language.

To hook packet processing, we manipulate functions related to packets. The functions hooked in BDSX are as follows:

-   `NetworkSystem::_sortAndPacketize` - `events.packetRaw`, `events.packetBefore`, `events.packetAfter`
-   `NetworkSystem::sendToMultiple`, `NetworkSystem::send`, `NetworkSystem::_sendInternal` - `events.packetSend`, `events.packetSendRaw`

`NetworkSystem::_sortAndPacketize` is a function that handles packets received by the server from the client. This function is hooked to implement `events.packetRaw`, `events.packetBefore`, and `events.packetAfter`.

`NetworkSystem::sendToMultiple` and `NetworkSystem::send` are functions used to send packets from the server to the client. These functions are hooked to implement `events.packetSend`. Since BDS uses `NetworkSystem::sendToMultiple` if multiple players are connected to the server and `NetworkSystem::send` if not, both functions must be hooked.\
`NetworkSystem::_sendInternal` is hooked to implement `events.packetSendRaw`.

Every time any packet is received by the server from the client, it goes through `events.packetRaw`, `events.packetBefore`, and `events.packetAfter` in that order. The test code for this is written in `example_and_test/test.ts`.\
Of course, not all packets always go through JS callbacks. Due to performance issues, packets pass through the assembly code described later, filtering out unnecessary packets.

```ts
// example_and_test\test.ts
packetEvents() {
    // ...
    for (let i = 0; i < 255; i++) {
        if (tooHeavy.has(i)) continue;
        events.packetBefore<MinecraftPacketIds>(i).on(
            this.wrap((ptr, ni, packetId) => {
                // ...
                this.equals(nextPacketPhase, PacketPhase.Before, `unexpected phase, id=${packetId}`);
                nextPacketPhase = PacketPhase.After;
                this.assert(nextEventTimeout !== null, "no timeout");
                clearTimeout(nextEventTimeout!);
                nextEventTimeout = setTimeout(() => {
                    this.error("packet after did not fire, id=" + packetId);
                }, 3000);
                this.assert(ni.getAddress() !== "UNASSIGNED_SYSTEM_ADDRESS", "packetBefore, Invalid ni, id=" + packetId);
                this.equals(packetId, idcheck, `packetBefore, different packetId on before. id=${packetId}`);
                this.equals(ptr.getId(), idcheck, `packetBefore, different class.packetId on before. id=${packetId}`);
            }, 0),
        );
        events.packetAfter<MinecraftPacketIds>(i).on(
            this.wrap((ptr, ni, packetId) => {
                // ...
                this.equals(nextPacketPhase, PacketPhase.After, `unexpected phase, id=${packetId}`);
                nextPacketPhase = PacketPhase.Raw;
                this.assert(nextEventTimeout !== null, "no timeout");
                clearTimeout(nextEventTimeout!);
                nextEventTimeout = null;
                this.assert(ni.getAddress() !== "UNASSIGNED_SYSTEM_ADDRESS", "packetAfter, Invalid ni, id=" + packetId);
                this.equals(packetId, idcheck, `packetAfter, different packetId on after. id=${packetId}`);
                this.equals(ptr.getId(), idcheck, `packetAfter, different class.packetId on after. id=${packetId}`);
            }, 0),
        );
        // ...
    }
}
```

## Packet Filtering

Running scripts for all packets would be a performance burden, so events should be processed only for packets with registered callbacks. This is implemented using an array called `enabledPacket` and bit flags.

```asm
; bdsx\asm\asmcode.asm
export def enabledPacket:byte[PACKET_ID_COUNT]
```

```ts
// bdsx\event.ts
const enabledPacket = asmcode.addressof_enabledPacket;
enabledPacket.fill(0, PACKET_ID_COUNT);
```

`enabledPacket` is a BYTE array of length 256, and `enabledPacket[i]` contains information about registered callbacks for the packet with ID `i` (`enum MinecraftPacketIds`).

There are five types of packet events: `Raw`, `Before`, `After`, `Send`, and `SendRaw`, each converted to a bit flag by the method like `const bit = 1 << PacketEventType.Before`. So each event's flag is `0b1`, `0b10`, `0b100`, `0b1000`, `0b10000`.\
For example, if `TextPacket` uses only `events.packetBefore` and `events.packetSend`, `enabledPacket[9] = 0b00001010`.

```ts
// bdsx\event.ts
export enum PacketEventType {
    Raw,
    Before,
    After,
    Send,
    SendRaw,
}
```

The following code is an algorithm applied commonly in the assembly for `packet*Hook`.

```asm
; bdsx\asm\asmcode.asm
    ; ...
    lea r10, enabledPacket
    mov ecx, eax
    mov al, byte ptr[rax+r10]
    test al, 0x08
    jz _pass
    ; ...
```

Currently, the `eax` register stores the packet ID. `byte ptr[rax+r10]` is equivalent to `enabledPacket[id]`, and the result is stored in the `al` register. The operation `test` is performed with `0x08`. The `test` instruction performs an `AND` operation between the two operands. Since `0x08` equals `0b1000`, this code jumps to the `_pass` section if the incoming packet isn't used in `events.packetSend`, executing the original code instead of the scripts. This is part of the `packetSendAllHook` code in `asmcode.asm`, and the same principle applies to `packetRawHook`, `packetBeforeHook`, etc.

## Structure of Stack

`rbp` register represents the base of the current stack frame, and `NetworkSystem::_sortAndPacketizeEvents` accesses to some values through `rbp`. the structure is defined as `OnPacketRBP` class.

```ts
// bdsx\event_impl\packetevent.ts
@nativeClass(null)
class OnPacketRBP extends AbstractClass {
    // NetworkSystem::_sortAndPacketizeEvents before MinecraftPackets::createPacket
    @nativeField(int32_t, 0x1a0)
    packetId: MinecraftPacketIds;
    // NetworkSystem::_sortAndPacketizeEvents before MinecraftPackets::createPacket
    @nativeField(CxxSharedPtr.make(Packet), 0x1a8)
    packet: CxxSharedPtr<Packet>; // NetworkSystem::_sortAndPacketizeEvents before MinecraftPackets::createPacket
    @nativeField(ReadOnlyBinaryStream, 0x260)
    stream: ReadOnlyBinaryStream; // after NetworkConnection::receivePacket
}
```

`0x1a0`, `0x1a8` is also used in assembly code: [packetRawHook](#packetrawhook), [packetBeforeHook](#packetbeforehook), [packetAfterHook](#packetafterhook)

## Hooking the Packet Process

In the hook, original codes should be executed. You can just copy-paste the assembly codes of the part used to patch(`procHacker.patching`).

### packetRawHook

1. Store the `NetworkConnection` instance of the client in `lastSenderNetId`. Other packet events use this to recognize the sender.
2. Do the packet filtering described earlier.
3. Call `onPacketRaw`, a function written in JS, to execute the script or execute the original code `createPacketRaw`.
4. For `packetRawHook`, the original code to be executed is calling `createPacketRaw`, which is either done in `onPacketRaw` or by skipping `onPacketRaw` to execute the original code.
5. In `packetRawHook`, event cancellation is implemented by making `onPacketRaw` return `null`, failing to create a packet.

```asm
; bdsx\asm\asmcode.asm
export def onPacketRaw:qword
export def createPacketRaw:qword
export def enabledPacket:byte[PACKET_ID_COUNT]
export def lastSenderNetId:qword

export proc packetRawHook
    ; dword ptr[rbp+0x1A0] - packetId
    mov lastSenderNetId, r14 ; NetworkConnection

    mov edx, dword ptr[rbp+0x1A0]
    lea rax, enabledPacket
    mov al, byte ptr[rax+rdx]
    unwind
    test al, 0x01
    jz _skipEvent
    mov rcx, rbp ; rbp
    mov rdx, r14 ; NetworkConnection
    jmp onPacketRaw
 _skipEvent:
    ; rdx - packetId
    lea rcx, [rbp+0x1A8] ; packet
    jmp createPacketRaw
endp
```

### packetBeforeHook

1. Execute the original code first.
2. Do the packet filtering described earlier.
3. Call `onPacketBefore`, a function written in JS, to execute the script.
4. Event cancellation in `packetBeforeHook` manipulates the return address directly, implemented in `onPacketBefore`.

```asm
; bdsx\asm\asmcode.asm
export def packetBeforeOriginal:qword
export def onPacketBefore:qword
export proc packetBeforeHook
    ; dword ptr[rbp+0x1A0] - packetId
    stack 28h
    lea rdx,qword ptr[rbp+0x2C0] ; original code
    lea rcx,qword ptr[rbp+0x48] ; original code
    call packetBeforeOriginal ; original code
    unwind
    lea rcx, enabledPacket
    mov r8d, dword ptr[rbp+0x1A0] ; packetId
    movzx ecx, byte ptr[rcx+r8]
    test cl, 0x02
    jz _skipEvent
    mov rcx, rbp
    mov rdx, rsp
    ; r8 - packetId
    mov [rbp+0x280], 0x1 ; assigns the correct value manually to bypass crashes - 1.20.61
    jmp onPacketBefore
_skipEvent:
    ret
endp
```

### packetAfterHook

1. Execute the original code first.
2. Do the packet filtering described earlier.
3. Call `onPacketAfter`, a function written in JS, to execute the script.
4. Since `packetAfter` is executed after the packet has already been received by BDS, event cancellation means simply not executing other callbacks.

```asm
; bdsx\asm\asmcode.asm
export def onPacketAfter:qword
export def handlePacket:qword
export def __guard_dispatch_icall_fptr:qword

export proc packetAfterHook
    ; dword ptr[rbp+0x1A0] - packetId
    stack 28h

    ; orignal codes
    mov r8, rsi ; callback
    mov rdx, r14 ; NetworkConnection
    mov rax, [rax+8]
    call __guard_dispatch_icall_fptr ; ServerNetworkHandler::handle()

    lea r10, enabledPacket
    mov r8d, dword ptr[rbp+0x1A0] ; packetId
    movzx eax, byte ptr[r10+r8]
    unwind
    test al, 0x04
    jz _skipEvent
    mov rcx,[rbp+0x1A8] ; packet
    mov rdx, r14 ; ni
    ; r8 - packetId
    jmp onPacketAfter
_skipEvent:
    ret
endp
```

### packetSendHook

1. Do the packet filtering described earlier.
2. Call `onPacketSend`, a function written in JS, to execute the script. the values ​​in the registers used to call it are backed up on the stack before the call, and then restored after the call.
3. Depending on whether the event is canceled which `onPacketSend` returns, executes the original codes or ends the event.

```asm
; bdsx\asm\asmcode.asm
export def sendOriginal:qword
export def onPacketSend:qword
export proc packetSendHook
    stack 48h

    mov rax, [r8] ; packet.vftable
    call [rax+8] ; packet.getId(), just constant return

    lea r10, enabledPacket
    mov r10b, byte ptr[rax+r10]
    test r10b, 0x08
    jz _skipEvent

    mov [rsp+20h], rcx
    mov ecx, eax ; packetId
    mov [rsp+28h], rdx
    mov [rsp+30h], r8 ; packet
    mov [rsp+38h], r9
    ; ecx = packetId
    call onPacketSend
    mov rcx, [rsp+20h]
    mov rdx, [rsp+28h]
    mov r8, [rsp+30h]
    mov r9, [rsp+38h]
    test eax, eax
    jnz _skipSend
_skipEvent:
    unwind
    jmp sendOriginal
_skipSend:
    unwind
    ret
endp
```

### packetSendAllHook

1. Do the packet filtering described earlier.
2. Call `onPacketSend`, a function written in JS, to execute the script.
3. Depending on whether the event is canceled which `onPacketSend` returns, executes the original codes or ends the event.

```asm
export def sendInternalOriginal:qword
export def packetSendAllCancelPoint:qword

export proc packetSendAllHook
    stack 28h
    ; r12 - packet
    ; rbx - ni

    mov rax, [r12]
    call [rax+8] ; packet.getId(), just constant return

    lea r10, enabledPacket
    mov ecx, eax
    mov al, byte ptr[rax+r10]
    test al, 0x08
    jz _pass

    mov r8, r12 ; packet
    mov rdx, rbx ; ni
    ; rcx = packetId
    call onPacketSend

    test eax, eax
    jz _pass
    unwind
    pop rax
    jmp packetSendAllCancelPoint
_pass:

    unwind
    ; original codes
    mov rax, [r12]
    mov rcx, r12
    movzx edi, byte ptr[rbx+0xa0]
    movzx esi, byte ptr[rcx+0x10]
    mov rax, qword ptr[rax+0x8]
    jmp __guard_dispatch_icall_fptr
endp
```

### packetSendInternalHook

1. Do the packet filtering described earlier.
2. Call `onPacketSendInternal`, a function written in JS, to execute the script. the values ​​in the registers used to call it are backed up on the stack before the call, and then restored after the call.
3. Depending on whether the event is canceled which `onPacketSendInternal` returns, executes the original codes or ends the event.

```asm
export def onPacketSendInternal:qword
export proc packetSendInternalHook
    stack 48h

    mov rax, [r8] ; packet.vftable
    call [rax+8] ; packet.getId(), just constant return

    lea r10, enabledPacket
    mov r10b, byte ptr[rax+r10]
    test r10b, 0x10
    jz _skipEvent

    mov [rsp+20h], rcx
    mov ecx, eax ; packetId
    mov [rsp+28h], rdx
    mov [rsp+30h], r8
    mov [rsp+38h], r9
    call onPacketSendInternal
    mov rcx, [rsp+20h]
    mov rdx, [rsp+28h]
    mov r8, [rsp+30h]
    mov r9, [rsp+38h]
    test eax, eax
    jnz _skipSend
_skipEvent:
    unwind
    jmp sendInternalOriginal
_skipSend:
    unwind
    ret
endp
```
