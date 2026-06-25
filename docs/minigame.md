# Minigame — the client model of a server board game

The 1.12.1 client carries a small, mostly-forgotten subsystem (`Source\Minigame\Minigame_C.cpp`): the
client-side model of a **server-authoritative turn-based board game**. The one registered type is
**TicTacToe**. The client holds the board and relays moves; the **server** decides whose turn it is
and who has won — the client does no game simulation of its own.

## The object

A singleton at `0xb71e5c`; the game object is `0x70` bytes:

| Offset | Field |
|---|---|
| `+0x10`, `+0x14` | game-id pair |
| `+0x18`, `+0x20` | player-name strings |
| `+0x28` | turn byte |
| `+0x29` | winner byte |
| `+0x2a` | the **9-cell board** (`0xFF` = empty) |

The class vtable is `0x8126b4` (slots `0x728700` / `0x728740` / `0x728770`). The subsystem occupies
`[0x7284a0, 0x728f00)`.

## The one piece of client-side logic: move legality

The only real computation the client owns is the integer move-legality check
(`minigame_move_legal`, `0x728b80`):

```
legal(idx)  =  idx in bounds
            && board[idx] == 0xFF        # the cell is empty
```

with the cell geometry derived as `row = idx / 3`, `col = idx % 3`. Everything else — turn order, win
detection — is the server's.

## Wire protocol

State is exchanged with the server over three opcodes:

| Opcode | Direction | Purpose |
|---|---|---|
| `SMSG 0x2f6` | server → client | setup |
| `SMSG 0x2f7` | server → client | state |
| `CMSG 0x2f8` | client → server | move |

The serialized state is the nine board cells plus the turn byte. Incoming packets are handled by the
Script/net glue: `HandleMinigameSetupPacket` (`0x4c4c70`, constructs and deserializes) and
`HandleMinigameStatePacket` (`0x4c4cf0`). UI updates are signalled through FrameScript events `0x216`
/ `0x217`.

## Lua surface

The UI drives it through three bindings: `Script::GetMinigameType`, `Script::MakeMinigameMove`, and
`Script::GetMinigameState` (`0x4c4d50` / `0x4c4d80` / `0x4c4df0`). There is also a separate world
game-object class for the in-world board, `CGGameObject_C_Type_MiniGame` (`0x5f6cb0`), in the
[object model](object-model.md).
