# shader-census

Mede, etapa por etapa, a tradução de shaders do Xbox Series (RDNA2/GFX10) para DXIL feita pelo
`victory-shader`: um registro JSONL por programa nativo com o resultado de IR, recursos, I/O,
estruturação, emissão DXIL, **validador DXIL** e criação de pipeline D3D12. Panics viram dado. O
`hash` é o mesmo que o VIctory imprime em runtime. Não contém nada do jogo; roda sobre os seus arquivos.

```
cd tools/shader-census
cargo build --release
target/release/shader-census censo.jsonl <seu>/shaders.rpf --validate --pipelines
```

Opções: `--validate` (precisa do `dxil.dll` do Windows SDK), `--pipelines`, `--jobs N`,
`--only hash,hash`, `--dump pasta` (DXIL emitida e texto completo do validador).

Relatórios (primeira etapa que falha, matriz por opcode), comparação do decoder com o LLVM e o mapa
efeito → programas ficam em scripts Python no repositório
[Rafa23xz/victory-shader-census](https://github.com/Rafa23xz/victory-shader-census): a política de
assets deste repositório não aceita `.py`.

---

Measures, stage by stage, how `victory-shader` translates Xbox Series shaders to DXIL: one JSONL record
per native program (IR, resources, I/O, structurisation, DXIL emission, DXIL validator, D3D12 pipeline),
panics isolated, `hash` as VIctory logs it. Report scripts live in
[Rafa23xz/victory-shader-census](https://github.com/Rafa23xz/victory-shader-census).
