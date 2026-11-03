---
description: schematize-zig — audita/scaffolda o <projeto>_ops (interface única) e verifica o fluxo de ambientes, a instalação paralela (nproc) e a independência dos serviços
argument-hint: "[bootstrap | audit | install]"
---

Governe a operação do sistema pelo **`<projeto>_ops`** (`references/ops.md`). Plan-first: **audita, mostra o plano (dry-run), pede aprovação, então chama a operação NOMEADA do ops** — e **nada toca o servidor fora do ops**. A IA **nunca** executa `ssh` mutável, `systemctl`/`docker`, console da app nem escreve SQL/comando próprio na VPS (piso 24): só pede operação tipada do ops; faltou operação, implemente-a no repo do ops via git.

## 1. Fluxo de ambientes (verifique primeiro)
Confirme que o pipeline força **dev local → teste local → GitHub → hml → prd** e que **nada vai direto pra hml/prd**:
- Deploy é **só a partir do git** (piso 23): o ops, rodando na VPS, faz `fetch` + `checkout` do **SHA/tag exato verificado**. **VETADO** `scp`/`rsync`/`tar | ssh`/`docker save|load`/`git bundle` e subir artefato montado fora do git; sem origem no git, é recusado.
- hml/prd com filesystem **read-only** (sem edição manual); **drift detection** ligado (o ops recusa/alerta divergência com o git).
- **VETADO editar código direto no servidor.** Achou edição manual → trate como incidente (archive), não como fluxo.

## 1.5. Layout, seed e redeploy destrutivo
Verifique/scaffolde (`schematize-engineering` -> `references/ops.md` §2):
- O ops provisiona em **`/<app>/`** e **clona os repos dentro** (`/<app>/<app>_<ctx>`, ex. `/payle/payle_core`). Repo fora de `/<app>/` é achado.
- **`/<app>/.env` é o seeder global** — fonte única de config; nenhum serviço com config à parte fora do seed.
- **Redeploy é destrutivo na app:** `ops redeploy` apaga a implantação anterior e recria um clone **zerado** só com o seed (sem patch in-place, sem drift). Prove idempotência: dois redeploys seguidos → mesmo estado.
- **Dados NUNCA no destrutivo:** banco/volumes/uploads preservados (migration reversível). Apagar dado é `ops reset`, **gated a dev/hml**, com confirmação — nunca prd.

## 1.6. Isolamento por usuário
Verifique/scaffolde (`schematize-engineering` -> `references/ops.md` §3):
- **Um user Linux dedicado por serviço** (`payle_core`→user `payle_core`), nunca dois no mesmo user, nunca `root`; a pasta do repo pertence ao user.
- **Um systemd unit hardened por serviço** (`User=`, `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `ReadWritePaths` mínimo). Comprometer um serviço não alcança os outros nem o host.
- **Tudo criado pelo ops** (user, permissão, unit) — nunca à mão.

## 2. O ops é a interface única (100%)
Verifique/scaffolde a CLI do `<projeto>_ops` com comandos **idempotentes**, `--help` e saída machine-readable, cobrindo **todo** o ciclo sem depender da IA:
`bootstrap` · `install`/`up` · `update` · `config` · `migrate` (reversível) · `health`/`doctor` · `rollback` · `logs`/`troubleshoot` · `test` (test kit a `schematize-qa` (test kit, `references/execucao.md` secao 2)).
- Se alguma operação de servidor **não** tem comando de ops, o gap é o achado: **crie o comando** — não faça por fora (`ssh` ad-hoc, editar arquivo, `docker`/`kubectl` na mão são vetados).
- Meta de completude: **`ops install` provisiona um servidor do zero** sozinho.

## 3. Instalação paralela (= nproc)
A instalação/subida roda **em paralelo**, grau = **`nproc`** (default; `--jobs N`/`OPS_JOBS` sobrepõe). Decomponha por **serviço independente** e dispare em ondas de `nproc`. Serialização só onde há dependência **real e declarada** (ex.: migração antes do consumidor), mínima e explícita.

## 4. Independência (invariante — prioridade máxima)
Rode a instalação **em paralelo de propósito** para provar independência. Se **qualquer** erro só acontece em paralelo (race, porta/lock/arquivo disputado, serviço que não sobe sem outro):
- **PARE a feature.** É bug de **independência de runtime** (piso 10 / §2), não de instalação.
- O ops **expõe** a colisão (relatório: o que disputou o quê, quem esperou quem). **Proibido serializar pra mascarar.**
- Corrija: cada serviço sobe **sozinho**, sem exigir outro no boot, com degradação graciosa (outbox/retry/fila). Só então o paralelo volta a `nproc`.

## 5. Saída
Grave o plano/relatório em `<projeto>_archive/` (§28): estado do fluxo de ambientes, cobertura de comandos do ops (o que falta), tempo de install e grau de paralelismo, e qualquer colisão de independência encontrada (com a correção priorizada). Confirme ao usuário: fluxo ok? ops cobre 100%? install paralela em `nproc`? independência provada?

## 6. Gate de saída (pisos 22/23)
Antes de dar por concluído: `git remote -v` presente, `git push` feito e `HEAD == origin/<branch>`; rode `bash .claude/skills/schematize-engineering/scripts/check-deploy-git.sh .` (reprova scp/rsync/tar|ssh/docker save|load em scripts, CI e docs). Local divergente do remoto = **não concluído**.
