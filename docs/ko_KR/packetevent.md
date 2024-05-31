# Exposing packets to JS

기본적으로 후킹은 기존 코드에 내가 원하는 코드를 실행하도록 하는 한 줄의 코드를 삽입하는 방식으로 진행됩니다. [코어 후킹](core.md)도 마찬가지입니다.\
그리고 BDSX는 core에서 제공하는, JS에서 포인터를 통하여 실제 메모리에 접근하도록 하는 API를 이용해 후킹을 구현합니다.

어셈블리 코드는 특별하지 않습니다. TS로 플러그인을 작성할 때처럼, 우리가 원하는 로직의 코드를 작성하면 됩니다. 단, 언어가 어셈블리일 뿐이죠.

패킷 처리를 후킹하려면 패킷과 관련된 함수를 조작하면 됩니다.
BDSX에서 후킹하는 함수는 다음과 같습니다.

-   `NetworkSystem::_sortAndPacketize` - `events.packetRaw`, `events.packetBefore`, `events.packetAfter`
-   `NetworkSystem::sendToMultiple`, `NetworkSystem::send`, `NetworkSystem::_sendInternal` - `events.packetSend`, `events.packetSendRaw`

`NetworkSystem::_sortAndPacketize`는 서버가 클라이언트로부터 받은 패킷들을 처리하는 함수입니다. 이 함수를 후킹해서 `events.packetRaw`, `events.packetBefore`, `events.packetAfter`를 구현합니다.

`NetworkSystem::sendToMultiple`, `NetworkSystem::send`는 서버가 클라이언트로 패킷을 보내는 함수입니다. 이 함수를 후킹해서 `events.packetSend`를 구현합니다. BDS가 서버에 여러 명의 플레이어가 접속하면 `NetworkSystem::sendToMultiple`을, 그렇지 않으면 `NetworkSystem::send`를 사용하여 패킷을 보내므로 두 함수를 모두 후킹해야 합니다.\
`NetworkSystem::_sendInternal`은 `events.packetSendRaw`를 구현하기 위해 후킹합니다.

어떤 패킷이든 서버가 클라이언트로부터 패킷을 받을 때마다 무조건 `events.packetRaw`, `events.packetBefore`, `events.packetAfter`를 이 순서대로 모두 거칩니다. 그리고 이에 대한 테스트 코드가 `example_and_test/test.ts`에 작성됐습니다.\
물론 항상 JS의 콜백을 거치는 것은 아닙니다. 성능 이슈로 정확히는 `events.packet*`이 아니라 후술할 어셈블리 코드를 거치며 여기서 필요 없는 패킷은 거릅니다.

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

모든 패킷에 대해 스크립트를 실행하기엔 성능에 부담이 되므로 콜백이 등록된 패킷에 대해서만 이벤트를 처리하도록 해야 합니다. 그리고 이는 `enabledPacket`이라는 배열과 비트 플래그를 통해 구현됩니다.

```asm
; bdsx\asm\asmcode.asm
export def enabledPacket:byte[PACKET_ID_COUNT]
```

```ts
// bdsx\event.ts
const enabledPacket = asmcode.addressof_enabledPacket;
enabledPacket.fill(0, PACKET_ID_COUNT);
```

`enabledPacket`은 256 길이의 BYTE 배열이며 `enabledPacket[i]`는 패킷 아이디(`enum MinecraftPacketIds`) 가 `i`인 패킷으로 등록된 콜백에 대한 정보입니다.

패킷 이벤트 콜백의 종류는 `Raw`, `Before`, `After`, `Send`, `SendRaw` 총 5가지이며 각각에 대한 비트 플래그는 `const bit = 1 << PacketEvnetType.Before`와 같은 방식으로 변환하여 사용합니다. 즉, 각 이벤트에 대한 플래그는 `0b1`, `0b10`, `0b100`, `0b1000`, `0b10000` 입니다.\
ex) `TextPacket`이 `events.packetBefore`, `events.packetSend`만을 사용한다면 `enabledPacket[9] = 0b00001010`이 됩니다.

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

다음은 어셈블리에서 `packet*Hook`에 공통으로 적용되는 알고리즘 입니다.

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

지금은 `eax` 레지스터에 packet id가 저장된 상황입니다. `byte ptr[rax+r10]`이 `enabledPacket[id]`와 동치인 상황이고 이를 `al` 레지스터에 저장한 후, `0x08`로 `test`를 수행합니다. `test` 명령어는 두 피연산자에 대해 `AND` 연산을 수행하고 `0x08` == `0b1000` 이므로, 이 코드는 들어온 패킷이 `events.packetSend`를 사용하고 있지 않다면 `_pass` 영역으로 `jump`하여 스크립트가 아닌 원래 코드를 실행하게 됩니다. 실제로 `asmcode.asm`에 있는 `packetSendAllHook` 코드의 일부이며, `packetRawHook`, `packetBeforeHook`... 도 같은 원리입니다.

## Hooking the packet process

### packetRawHook

1. `lastSenderNetId`에 저장함. 다른 패킷 이벤트에서 이걸 불러와서 주체를 결정함
2. 이후에 전술한 패킷 필터링을 수행함
3. 그후 `onPacketRaw`라는, JS로 작성된 함수를 호출하여 스크립트를 실행하거나 원래 실행해야할 코드인 `createPacketRaw`를 실행함.
4. `packetRawHook`의 경우 실행해야할 original code는 `createPacketRaw`를 호출하는 것인데, 이는 `onPacketRaw`에서 실행하거나 `onPacketRaw`를 스킵하고 original code를 실행함

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

1. 일단 오리지널 코드를 실행함. 이 오리지널 코드는 patch하느라 사용한 부분의 어셈블리 코드를 그대로 복붙하면 됨.
2. 이후에 전술한 패킷 필터링을 수행함
3. 그후 `onPacketBefore`라는, JS로 작성된 함수를 호출하여 스크립트를 실행함

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

1. original code를 실행함
2. 이후에 전술한 패킷 필터링을 수행함
3. 그후 `onPacketAfter`라는, JS로 작성된 함수를 호출하여 스크립트를 실행함

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

1. 전술한 패킷 필터링을 수행함
2. 그후 `onPacketSend`라는, JS로 작성된 함수를 호출하여 스크립트를 실행함. 이때 `onPacketSend`를 호출하는 데 사용할 레지스터에 있는 값들을 스택에 백업한 후, 호출하고 나서 다시 레지스터에 저장함.
3. 그리고 `onPacketSend`가 반환하는 이벤트 취소 여부에 따라 original code를 실행하거나 그냥 이벤트를 끝냄.

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

1. 전술한 패킷 필터링을 수행함
2. 그후 `onPacketSend`라는, JS로 작성된 함수를 호출하여 스크립트를 실행함.
3. 그리고 `onPacketSend`가 반환하는 이벤트 취소 여부에 따라 original code를 실행하거나 그냥 이벤트를 끝냄.

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

1. 전술한 패킷 필터링을 수행함
2. 그후 `onPacketSendInternal` 이라는, JS로 작성된 함수를 호출하여 스크립트를 실행함. 이때 `onPacketSendInternal`를 호출하는 데 사용할 레지스터에 있는 값들을 스택에 백업한 후, 호출하고 나서 다시 레지스터에 저장함.
3. 그리고 `onPacketSendInternal`이 반환하는 이벤트 취소 여부에 따라 original code를 실행하거나 그냥 이벤트를 끝냄.

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
