# Plugin Vencord no Discord Linux: conf própria e updater — 2026-10-17

Prova real no cliente oficial do Discord rodando no CachyOS (kernel `7.2.9-2-cachyos`), com o
build de desenvolvimento do Vencord (`~/Vencord`, userplugin em `src/userplugins/goLiveBypass`).
O alvo foram as duas correções do ciclo — modo **"Arquivo WireGuard personalizado"** e o
**journal do updater preso** — e o ciclo de atualização nos canais stable e beta ponta a ponta,
que é onde os relatos apontaram problema.

## Como foi montado

- Discord 1.0.161 (Electron 42.11.10) foi iniciado com `--remote-debugging-port=9222`; o bridge
  nativo do plugin (`window.VencordNative.pluginHelpers.GoLiveBypass`) foi chamado por CDP —
  **as mesmas funções que os botões do painel chamam**. Nenhum harness: processo, polkit,
  WireGuard e namespace são os reais.
- A conf do usuário saiu do próprio helper do projeto
  (`goLiveBypass/bin/linux-x64/proton-confgen -username … -session-file … -output …
  -json -ipv6 -exclude-countries BR -auto-ping -free-only`) usando a sessão Proton já salva
  pela GUI: `MX-FREE#12`, endpoint `84.20.27.50:51820`, `AllowedIPs = 0.0.0.0/0, ::/0`.
  O `.conf` que estava em `~/Downloads` tem a `PrivateKey` **redigida** (`*****`) e não ativa.
- A senha do polkit foi digitada pelo operador na janela de autorização (o ambiente não tem
  agente gráfico do polkit; o plugin cai no terminal da lista dele — Alacritty).
- Nenhuma chave privada, senha ou token de sessão foi registrado aqui.

## Resultado — conf manual no modo custom

| Verificação | Resultado |
|---|---|
| Importação da conf escolhida | `success:true path:…/plugin-vpn/custom-wireguard.conf`, arquivo `0600` e byte a byte igual à fonte |
| Rota registrada | `active-route.json` com `mode:"custom"`, `source:"imported"`, `endpoint:"84.20.27.50:51820"` |
| "Testar .conf" sem argumento | valida a conf **configurada**, não o perfil ativo: a primeira tentativa devolveu o erro da chave do arquivo escolhido, e não "nenhuma configuração encontrada" |
| Ativação | `success:true state:"restart_pending"` — namespace `glb-28x3175d9c74f` e interface `glbwg28x3175d9c` criados numa única autorização; Discord relançado dentro do namespace |
| Isolamento observado | processo principal do Discord em `net:[4026534053]` contra `net:[4026531833]` do host; `owner.lock` com o pid do processo em execução |
| Túnel conectado (root, dentro do namespace) | `wg show`: `latest handshake: 14 seconds ago`, `transfer: 27.54 KiB received, 40.56 KiB sent`, peer `84.20.27.50:51820`, `allowed ips: 0.0.0.0/0, ::/0`; rota `default dev glbwg28x3175d9c` |
| IP de saída | dentro do túnel `205.147.22.22` (Proton/MX) contra `200.222.95.x` do host — o tráfego do Discord sai pela VPN, não pela conexão local |
| Fila depois da ativação | o painel seguiu respondendo `getVpnStatus`/`getPluginUpdateStatus` nas consultas seguintes; a fila serial destravada tem prova de teste (`plugin-custom-config.test.ts`: ativação, `restoreNetwork()` e `shutdown` na mesma fila) |

## Resultado — updater (fork `pdl-clay/GoLiveBypass`)

Dois releases por canal, todos com SHA-256 conferido, `bug-report.ts` e
`vpn-snapshot-worker.ts` presentes e as correções dentro do zip:

| Etapa | Canal | Seleção | Journal / log | Resultado |
|---|---|---|---|---|
| 2.0.10 → 2.0.13 | beta | `2.0.13` (o maior disponível naquele momento) | `phase:"staged"` → `plugin 2.0.13 baixado e validado` → `plugin preparado de 2.0.10 para 2.0.13` | instalado `2.0.13` |
| 2.0.13 → 2.0.14-beta-1 | beta | `2.0.14-beta-1` (prerelease, > estável instalada) | `staged` → `baixado e validado` → `preparado de 2.0.13 para 2.0.14-beta-1` | instalado `2.0.14-beta-1` |
| 2.0.14-beta-1 → 2.0.14 | stable | `2.0.14` (estável mais nova, sem downgrade) | `staged` → `baixado e validado` → `preparado de 2.0.14-beta-1 para 2.0.14` | instalado `2.0.14` |
| 2.0.14 (estável) | stable | nenhuma (já é a mais nova) | `pending:false`, `lastError:null` | nada a fazer |

Em todas as etapas o painel terminou em `lastError:null`, sem journal no slot ativo, e os
diretórios `.golivebypass-update-staging/` e `.golivebypass-update-backups/` do checkout
ficaram vazios depois da troca (a troca e o `pnpm build` acontecem na abertura seguinte).

### Journal preso, reproduzido ao vivo

Com o plugin 2.0.13 em execução foi escrito um marcador `prepared` com `sourceDigest`
divergente da árvore (o mesmo cenário do relato "digest da árvore preparada não confere com a
fonte atual"). A consulta seguinte **não** devolveu erro: o marcador foi movido para
`plugin-update-pending.json.stale-source-…json`, o log registrou `motivo:"stale-source"` e o
status ficou `pending:false, lastError:null` — antes disso a mensagem ficava presa e toda
consulta falhava igual.

## Limites desta prova

1. Os releases deste ciclo foram publicados no **fork de teste** (`pdl-clay/GoLiveBypass`) a
   partir da branch `release/update-e2e-2.0.11`, que aponta upstream/owner/REPO para o fork;
   nada foi publicado em produção (`bezumiya/GoLiveBypass`).
2. A prova do handshake e do IP de saída foi feita com `sudo ip netns exec … wg show` e um
   `curl` dentro do namespace (root, executado pelo operador); o plugin em si não mede isso no
   Linux — os números estão na tabela acima, não na interface.
3. Os reinícios do Discord no ciclo de update foram `SIGKILL` + nova execução a partir do host
   (um encerramento normal restaura a rede, por desenho, e derrubaria o túnel em teste). Nessas
   condições o plugin classifica a sessão como `recovery_required` — namespace ativo com o
   processo atual fora dele — e **não** adota o túnel sem ação explícita: comportamento
   observado, não defeito. No fim do ciclo o namespace foi removido com a mesma sequência de
   rollback do plugin (`ip link del`, `ip netns del`, `rm -rf /etc/netns/<ns>`), os registros
   `owner.lock`/`active-route.json` foram apagados e a abertura seguinte do Discord confirmou
   `VPN inactive | ativa false | ownership false`; a rede do host nunca foi alterada
   (`default via 192.168.100.1 dev enp3s0`, `resolv.conf` do NetworkManager intactos).
4. O `plugin-vpn.log` só começou a registrar quando o arquivo foi criado à mão: a instalação
   nova não criava o arquivo (defeito separado, corrigido em `fix(plugin): log nasce na
   primeira escrita`, ainda **fora** dos artefatos 2.0.13/2.0.14 publicados no fork).
5. O `.conf` de `~/Downloads` do usuário está com a chave redigida; a conf válida usada no
   teste foi gerada na hora pelo helper com a sessão Proton salva.
