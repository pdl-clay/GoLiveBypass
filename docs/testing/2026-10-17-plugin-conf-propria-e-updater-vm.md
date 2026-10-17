# Plugin: conf própria e journal do updater na VM Windows — 2026-10-17

Prova real na VM Windows 11 (`WinBoat`, `wvm`) das duas correções do plugin deste ciclo:
o modo "Arquivo WireGuard personalizado" que não carregava a conf e o journal do updater que
travava toda consulta com "digest da árvore preparada não confere com a fonte atual".

## Como foi montado

A VM não tem Discord/Vencord instalado, então o teste não passou pela interface do cliente:
o **código real do plugin** foi empacotado com `esbuild` (`--bundle --format=cjs
--platform=node --external:electron`) e executado sob **Node 24 no Windows**, com dois stubs
— `electron` (só `app`/`ipcMain`/`dialog`, `whenReady` que nunca resolve e `relaunch`/`exit`
inertes) e `@main/settings` (`RendererSettings`). Todo o resto é real: `vpn-controller.ts`,
`vpn-windows.ts`, o snapshot do PowerShell, o download e a instalação do WireSock SDK, o
serviço `wiresock-client-service`, o data dir em `%LOCALAPPDATA%` (no harness do updater) e
`C:\golive-vmtest\data\plugin-vpn` (no harness da conf).

Os dois harnesses ficaram em `C:\golive-vmtest` na VM (junto dos bundles), e a conf de teste
usa um par de chaves X25519 gerado na hora e o endpoint público `162.159.192.1:2408` — o
túnel sobe com o perfil do usuário, sem handshake com peer real (não era o alvo).

## Resultado — conf própria (`conf-harness.mjs`, 37 s, `TODOS OS CHECKS PASSARAM`)

| Verificação | Resultado |
|---|---|
| import da conf grava a cópia própria | `custom-wireguard.conf` com o conteúdo importado |
| ativação no modo custom conclui | `success:true state:restart_pending` em **6,7 s** (antes: travava em "Preparando perfil e WireGuard") |
| perfil aplicado é o do usuário | `wiresock-discord.conf` com `Endpoint = 162.159.192.1:2408` e `#@ws:AllowedApps` |
| serviço Windows com o perfil do plugin | `sc qc`: `wiresock-client.exe service -config "…\plugin-vpn\wiresock-discord.conf"`; `sc query`: `ESTADO: 4 RUNNING` |
| restauração conclui | `success:true state:inactive` (a fila continua viva depois da ativação) |
| logout Proton não apaga a conf do usuário | cópia própria intacta depois de `logoutProton()` com perfil Proton em uso |
| ativação custom depois do logout | aplica o endpoint do usuário e **não** o do perfil Proton |
| layout antigo (conf apontando para o perfil Proton) | recusado: "A configuração personalizada aponta para o perfil Proton em uso. Selecione o arquivo .conf novamente." |
| "Testar .conf" sem conf configurada | falha com "Nenhum arquivo WireGuard personalizado foi importado ainda." |

Estado final da VM conferido fora do harness: `sc query wiresock-client-service` sem serviço e
nenhum processo `wiresock` — a restauração removeu o túnel.

## Resultado — updater (`updater-harness.mjs`, `TODOS OS CHECKS PASSARAM`)

Checkout falso em `C:\golive-vmtest\checkout` (árvore real do plugin, com
`bin\win32-x64\proton-confgen.exe`) e journals reais em
`%LOCALAPPDATA%\GoLiveBypass\plugin-vpn`, com `getPluginUpdateStatus` — o mesmo caminho que
alimenta o painel:

| Journal | Antes | Agora |
|---|---|---|
| `prepared` com digest divergente | `lastError` = "digest da árvore preparada não confere…", journal preso no slot ativo | `lastError: null`, quarentena `plugin-update-pending.json.stale-source-…json` |
| `staged` sem a fonte em staging | `lastError` = "update baixado não tem mais a fonte em staging…", preso | `lastError: null`, quarentena `…missing-staging-…json` |
| `prepared` legado sem digest | quarentenado (já era) | quarentena `…legacy-…json` |
| `preparing` (promoção interrompida) | resolvido pela recuperação | inalterado: recuperação resolve, sem quarentena nova |

## Limites desta prova

- Não passou pela interface do Vencord/Discord: o harness chama o controlador e o handler do
  updater direto. O painel só muda por consumir o mesmo `getPluginUpdateStatus`.
- `app.relaunch`/`exit` são stub: o estado fica em `restart_pending` (o normal de um "Ativar
  agora" explícito) e `active` só viraria `true` depois do relaunch real.
- `AllowedApps` do perfil sanitizado lista `node.exe` (o harness roda sob Node), não
  `Discord.exe`; o mecanismo do filtro por aplicativo é o mesmo.
- Sem handshake com peer WireGuard real (endpoint público, chaves novas): a prova é do
  fluxo/serviço, não de conectividade nem de rota geográfica.
- Duas linhas de aviso aparecem na restauração — `operação elevada do WireSock falhou` e
  `não consegui encerrar processo WireSock próprio`. São a corrida benigna do `taskkill` de
  um pid que a parada do serviço já recolheu (`killProcesses` → `runAsAdministrator`); o
  `Start-Process -Verb RunAs` funciona nesta sessão (verificado com um probe) e o serviço foi
  removido de fato. Ficou registrado, não é defeito das correções deste ciclo.
- O primeiro ensaio sofreu uma reconexão do SSH que repetiu o comando: o harness apagava o
  data dir com o serviço ainda ativo e adotou o túnel anterior. A versão atual derruba
  qualquer túnel herdado antes de limpar o diretório, o que tornou a execução idempotente.
