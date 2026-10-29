# Compilação e validação nativa ARM64

Compilação concluída em 2026-10-29, branch deploy/tinker2-cwb, fonte e9bd4321.
Toolchain oficial Go 1.27.3 linux/arm64, instalado em prefixo isolado.
Arquivo go1.27.3.linux-arm64.tar.gz verificado por SHA256:
a06e99309dbcbdca1a8ccd2282c3ee146e55930886e26b4f1193269587165f49.
Não foi substituído um Go global nem atualizada a distribuição.

Comando de build: make -j1 build-server build-ingestor, com GO apontando ao
binário isolado, CGO_ENABLED=1, GOMAXPROCS=2, GOMEMLIMIT=512MiB,
GO_BUILD_FLAGS='-trimpath -p=1' e nice 10. Tags nativas do Makefile:
netgo,osusergo,sqlite_omit_load_extension. Caches de módulos/compilação separados.

## Artefatos locais

- dist/corescope-server: ARM64 ELF, ~15 MiB, SHA256
  c113cc88a8e2ae148c1a90fee257b75c36a3cd6132b3ca6ec5746bb445895de9.
- dist/corescope-ingestor: ARM64 ELF, ~9,2 MiB, SHA256
  dd3f4fb837cadb5e5ab7b26f6b64b53595da622a10279553626c58c6d652652e.

Binários nativos ligados dinamicamente à libc do sistema; não são os binários
estáticos musl do Docker. Os artefatos não foram adicionados ao histórico Git.

## Teste de execução

Instância temporária, banco próprio em RAM, configurações/credenciais somente
locais fora do repositório. Ambos os processos usaram os GOMEMLIMIT do perfil
(192 MiB servidor, 96 MiB ingestor) e GOMAXPROCS=2.

- Ingestor abriu SQLite real e inicializou 26 tabelas; integrity_check=ok.
- Conectou ao broker MQTT local e recebeu status CWB real.
- /api/healthz respondeu HTTP 200, ready=true, com liveness da fonte CWB-local.
- / respondeu HTTP 200 e HTML da interface estática.
- Ambos permaneceram ativos durante a verificação e encerraram ao fim do teste.
- Nenhum serviço CoreScope permanente foi instalado/ativado.

O teste comprova a integração CGO/SQLite, inicialização e HTTP/MQTT. Não é teste
de carga, validação visual em navegador ou comprovação do teto de cgroup. Não
foi observada transmissão RF nesse teste curto. A aplicação dos limites systemd
continua pendente da instalação das units. A configuração do perfil deve ser
copiada com credenciais reais apenas para o ambiente local na implantação.
