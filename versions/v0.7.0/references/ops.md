<!-- cross-skill: references/orquestracao.md -> schematize-engineering -->

# Operação pelo ops — ambientes, instalação e correção (control plane)


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/ops.md`. Leia lá primeiro; aqui fica **só o que muda em Zig**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

> O **`<projeto>_ops`** é a **interface única** de operação do sistema. Invariantes desta reference, todos **INEGOCIÁVEIS**: (1) nada chega ao servidor sem passar pelo **fluxo de promoção** (dev local → teste local → GitHub → hml → prd); (2) o ops provisiona num **workspace por aplicação** (`/<app>/`) e todo **redeploy é destrutivo na aplicação, semeado pelo `/<app>/.env` global** — mas **nunca destrói dados**; (3) cada serviço roda **isolado por usuário** (user Linux + systemd hardened); (4) **100%** de instalação/atualização/correção/config passa por uma **ferramenta do ops** — nunca à mão; (5) instalar é **paralelo por padrão** (= `nproc`), e falha no paralelo = **bug de independência** (prioridade máxima). Contexto do ops: `arquitetura.md` §2. Deploy/ambientes: `schematize-engineering` -> `operacao.md` §21. Test kit: `testes.md` a `schematize-qa` (test kit, `references/execucao.md` secao 2).

## 4. O ops é a interface única (100% das operações)

- **Superfície mínima** (idempotentes, com `--help` e saída machine-readable): `bootstrap` (cria `/<app>/` e clona os repos) · `install`/`up` · `redeploy` (destrutivo, do seed §2) · `update` · `config` (do `/<app>/.env`) · `migrate` (reversível) · `health`/`doctor` · `rollback` · `logs`/`troubleshoot` · `reset` (destrói dados — gated, dev/hml) · `test` (ver a `schematize-qa` (test kit, `references/execucao.md` secao 2)).

## 8. Pisos 22–24 — recorte zig

- **Canal único = git (22/23); o ops é o gate da IA (24).** Detalhe na base (`schematize-engineering` → `references/ops.md` §1, §4, §8): sem `scp`/`rsync`/`tar | ssh`/`docker save|load`; a IA nunca muda a VPS nem escreve SQL — só operação nomeada do ops; antes de pronto, `HEAD == origin` (`scripts/check-deploy-git.sh`).
- **Zig:** binário cross-compilado na estação (`zig build -Dtarget=…`) e enviado por `scp`/`rsync` é **VETADO**; o ops, na VPS, busca o SHA/tag do git (`build.zig.zon` com hashes e `minimum_zig_version` pinados) e builda limpo. Estático/musl não muda a regra: o que sobe é build do git, nunca upload.

## 7. Integração com o resto da casa

Comando: **`/<slug>-ops`** (ex.: `/eng-ops`, `/zig-ops`) — audita/scaffolda o ops, verifica o fluxo de ambientes, o layout `/<app>/` + seed, o isolamento por usuário, a paralelização (`nproc`) e a independência.
