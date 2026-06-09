# fizz-memory-l1-ring

Fizz 記憶系の **L1**(最速層)。user_id ごとに直近 N 件の `MemoryEvent` を
インメモリのリングバッファで保持し、recall に即答する。LLM も I/O も無い hot path。

記憶は 3 層に分かれる:

| 層 | 部品 | 役割 |
|---|---|---|
| **L1** | **fizz-memory-l1-ring**(これ) | per-user 直近発言のインメモリ ring(~即時) |
| L2 | fizz-memory-l2-events | 全発言の永続化と検索 |
| L3 | fizz-memory-l3-summarizer | 閾値超で per-user 要約を rolling 更新 |

推論の hot path は L1 だけ見れば「同じ枠で直前に何を言ったか」が分かる。永続化・
要約は別部品の責務で、ここには持たない。

## I/O

stdin に 1 行 1 コマンド(NDJSON, `fizz_protocol.memory` の Codec):

```jsonc
{"op":"note","event":{"id":1,"user_id":7,"session_id":null,"role":"user","content":"こんにちは","ts":100,"tags":[]}}
{"op":"recall","user_id":7}
```

`note` はバッファに積むだけ(出力なし)。`recall` は直近 N 件を 1 行で返す:

```json
{"user_id":7,"recent":[{"id":1,"user_id":7,"role":"user","content":"こんにちは","ts":100,"tags":[]}]}
```

不正な行は stderr に警告して読み飛ばす(パイプは止めない)。EOF で終了。

## 動作

- バッファは user_id ごとに `cap` 件で頭打ち(古い順保持、超過は先頭=最古から drop)。
- `cap` は環境変数 `FIZZ_L1_CAP` で上書き(既定 16、下限 1)。
- 長時間配信のメモリ膨張を防ぐ trade-off — `cap` より前の発言は L1 からは消える(L2 には残る)。

## 開発

```sh
almide check src/main.almd
almide test
almide build src/main.almd -o build/fizz-memory-l1-ring
```

ツールチェーン: [almide](https://github.com/almide/almide) v0.26.6+。
WASM backend がコンパイラバグで落ちる場合の native fallback に rustc が必要。

## 契約

[fizz-protocol](https://github.com/Aid-On/fizz-protocol) の `memory` モジュール
(`MemoryEvent` / `MemoryRole` とその Codec)に依存。
