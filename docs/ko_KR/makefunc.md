# makefunc

`makefunc`는 BDSX에 구현된 JS에서 네이티브 함수를 사용하기 위한 모듈입니다.
`procHacker`의 함수들은 내부적으로 `makefunc.np`, `makefunc.js`을 통해 구현됩니다.

## makefunc.js

`makefunc.js`는 네이티브 함수를 JS 함수로 만들어주는 함수입니다.

## makefunc.np

`makefunc.np`는 JS 함수를 네이티브 함수로 만들어주는 함수입니다.

# procHacker

## procHacker.js, procHacker.np

`procHacker.js`, `procHacker.np` 각각 `makefunc.js`, `makefunc.np`와 목적과 역할이 같습니다. 다만 `procHacker`는 `makefunc`보다 더 노출되는 API 입니다.

## procHacker.patching

다음은 `procHacker.patching`의 시그니쳐 입니다.

```ts
// based on part of bdsx\prochacker.ts
patching(
        subject: string,
        key: Extract<keyof T, string>,
        offset: number,
        newCode: VoidPointer,
        tempRegister: Register,
        call: boolean,
        originalCode: (number | null)[],
        ignoreAreaOrOpts?: number[] | ProcHackerPatchOptions, // deprecated
    ): void
```

다음은 사용 예시 입니다.

```ts
// bdsx\event_impl\packetevent.ts
procHacker.patching(
    "hook-packet-after",
    packetizeSymbol,
    0x7b2,
    asmcode.packetAfterHook, // original code depended
    Register.rdx,
    true,
    // prettier-ignore
    [
            0x4C, 0x8B, 0xC6,                          // mov r8,rsi
            0x49, 0x8B, 0xD6,                          // mov rdx,r14
            0x48, 0x8B, 0x40, 0x08,                    // mov rax,qword ptr ds:[rax+8]
            0xFF, 0x15, null, null, null, null,        // call qword ptr ds:[<__guard_dispatch_icall_fptr>]
    ],
);
```

| parameter name | used value              | description                                                                                      |
| -------------- | ----------------------- | ------------------------------------------------------------------------------------------------ |
| `subject`      | "hook-packet-after"     | You can treat this as just a tag or label for debugging                                          |
| `key`          | packetizeSymbol         | It's the symbol of target function to hook                                                       |
| `offset`       | 0x7b2                   | It's offset to hook the target. sometimes we want to change only parts of the functions.         |
| `newCode`      | asmcode.packetAfterHook | It's the new code to use.                                                                        |
| `tempRegister` | Register.rdx            | It's the temp register to use when call the function. used with `jmp` or `call`.                 |
| `call`         | true                    | If this is true, `call` operation is used, if false then `jmp` is used.                          |
| `originalCode` | [...]                   | Before patching, checks the memory with this value. if they are unmatched then stop the process. |

`origianCode` is replaced with `newCode`. so we may need to run the original code manually in `newCode`.
