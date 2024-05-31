# ASM

BDSX는 어셈블리 코드를 직접 실행할 수 있습니다.\
내장된 자체 어셈블러로 메모리에 코드를 할당하는데, 해당 메모리 공간에는 어셈블리에서 선언한 함수와 변수들이 저장되어 있고, 이는 JS에서도 접근할 수 있으며 변수의 경우 쓰기도 가능합니다.\
내부적인 멀티스레드 후킹, JS 함수 <-> 네이티브 함수, 패킷 처리 시스템 후킹 등등의 여러 기능들이 어셈블리 코드로 구현되어 있습니다.

예를 들어보면\
다음은 패킷 후킹 코드의 일부입니다.
`packetevent.ts#L7`에서 사용된 `asmcode.lastSenderNetId`는 `asmcode.asm#L5`에서 선언된 qword 크기의 변수입니다.

이 단계에서는 코드의 구조를 정확히 이해할 필요는 없습니다. 단지 어셈블리 코드와 JS 코드가 소통 가능하다는 것을 알려주기 위한 페이지입니다. 일종의 FFI라고 생각하시면 됩니다.

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
