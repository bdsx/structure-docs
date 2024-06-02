# ASM

BDSX has the capability to directly execute assembly code.\
It includes a built-in assembler that allocates memory for the code, storing functions and variables declared in the assembly. These functions and variables can be accessed from JS, and particularly, variables can also be written to.\
The assembly code is used for implementing various functionalities such as internal multithreaded hooking, converting JS function <-> native function, and packet processing system hooking.

For instance,\
let's look at a portion of the packet hooking code. The `asmcode.lastSenderNetId` used in `packetevent.ts#L7` is a variable of the size of `qword` declared in `asmcode.asm#L5`.

At this step, it's not necessary to understand the code structure in detail. The main point is to illustrate that the assembly and JS can communicate with each other, functioning as a form of FFI.

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

```ts
// bdsx\event_impl\packetevent.ts
function onPacketBefore(rbp: OnPacketRBP, returnAddressInStack: StaticPointer, packetId: MinecraftPacketIds): void {
    try {
        const target = events.packetBefore(packetId);
        if (target === null || target.isEmpty()) throw Error("no listener but onPacketBefore fired.");

        const ni = nethook.lastSender || asmcode.lastSenderNetId.as(NetworkConnection).networkIdentifier;
        const TypedPacket = PacketIdToType[packetId] || Packet;
        const packet = rbp.packet.p!;
        const typedPacket = packet.as(TypedPacket);
        try {
            for (const listener of target.allListeners()) {
                try {
                    if (listener(typedPacket, ni, packetId) === CANCEL) {
                        returnAddressInStack.setPointer(packetBeforeSkipAddress);
                    }
                } catch (err) {
                    events.errorFire(err);
                }
            }
        } finally {
            decay(typedPacket);
        }
    } catch (err) {
        remapAndPrintError(err);
    }
}
```
