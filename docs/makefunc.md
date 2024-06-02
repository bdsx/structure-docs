# makefunc

`makefunc` is a module in BDSX designed to enable the use of native functions in JS. The functions in `procHacker` are internally implemented using `makefunc.np` and `makefunc.js`.

## makefunc.js

`makefunc.js` converts native functions into JS functions.

## makefunc.np

`makefunc.js` converts JS functions into native functions.

# procHacker

## procHacker.js, procHacker.np

`procHacker.js` and `procHacker.np` serve the same purpose and role as `makefunc.js` and `makefunc.np`, respectively. However, procHacker is a more exposed API compared to makefunc.

## procHacker.patching

Below is the signature for `procHacker.patching`.

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
        ignoreAreaOrOpts?: number[] | ProcHackerPatchOptions, //  deprecated
    ): void
```

Here is an example.

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

| parameter name | used value              | description                                                                                       |
| -------------- | ----------------------- | ------------------------------------------------------------------------------------------------- |
| `subject`      | "hook-packet-after"     | You can treat this as just a tag or label for debugging                                           |
| `key`          | packetizeSymbol         | It's the symbol of target function to hook                                                        |
| `offset`       | 0x7b2                   | It's offset to hook the target. sometimes we want to change only parts of the functions.          |
| `newCode`      | asmcode.packetAfterHook | It's the new code to use.                                                                         |
| `tempRegister` | Register.rdx            | It's the temp register to use when call the function. used with `jmp` or `call`.                  |
| `call`         | true                    | If this is true, `call` operation is used, if false then `jmp` is used.                           |
| `originalCode` | [...]                   | Before patching, checks the memory with this value. if they are unmatched then stops the process. |

`origianCode` is replaced with `newCode`. so we may need to run the original code manually in `newCode`.
