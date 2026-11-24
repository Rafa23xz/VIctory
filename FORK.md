# Este fork / This fork

Branch `rafa/shaders`: o `main` do [Marvisak/VIctory](https://github.com/Marvisak/VIctory) com PRs
abertos aplicados juntos, correções locais encontradas rodando o jogo numa RTX 4080 (NVIDIA, ondas de
32 lanes) e uma correção do tradutor de shaders. Créditos de cada PR na mensagem do primeiro commit.

| Commit | O quê |
|---|---|
| 1 | PRs #189 #269 #135 #205 #206 #89 #123 #305 #261 #432 #193 #136 + mudanças `local:` (sync do console vira barreira UAV, workers de pipeline sobrevivem a panic, copy-back por `NtWriteVirtualMemory`, user data além dos 16 SGPRs, UAV raiz limitada) |
| 2 | `v_readfirstlane` de um valor igual em toda lane é o próprio valor (emulação wave64 → wave32) |
| 3 | `tools/shader-census`: censo por etapa (IR, recursos, I/O, estruturação, DXIL, validador, pipeline) |

Medido no arquivo de shaders do jogo (26.084 programas, validador DXIL + pipeline no host):

| Estado | Passam todas as etapas | Compute | Panics no emissor |
|---|---:|---:|---:|
| `main` d2a4bae | 21.656 | 1.355 | 0 |
| commit 1 | 22.316 | 1.393 | 29 |
| commit 2 | 22.400 | 1.477 | 46 |

Problemas conhecidos (abertos): 785 programas emitidos são rejeitados pelo validador DXIL mas viram
pipeline (o contêiner é assinado aqui) — atômicos de imagem de 64 bits num recurso do heap tipado como
float4 e sem `AtomicInt64OnHeapResource`, LOD implícito em compute, saída de mesh acima de 32.768;
panic "forward reference outside a phi" no caminho emulado (o worker de pipeline o captura); a emulação
recusa `v_readfirstlane` em regiões onde só uma lane está ativa.

Nada do jogo está aqui (`cargo xtask policy` passa). O mundo ainda renderiza quase todo preto.

---

Upstream VIctory `main` plus open PRs applied together, local fixes found running the game on an
RTX 4080 (NVIDIA, 32-lane waves), and one shader translator fix. Per-PR credits are in the first
commit's message. Numbers and known issues above.
