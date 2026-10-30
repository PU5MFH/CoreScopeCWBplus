# CoreScopeCWBplus — perfil inicial para Tinker2 / ARM64 / 2 GB

Fork de Kpa-clawbot/CoreScope. Base examinada: ddce76a. Objetivo: executar
nativamente os binários Go server e ingestor, servindo os arquivos public/,
consumindo somente CWB de um broker MQTT local já existente.

O perfil prepara configuração e units. A compilação nativa e um teste temporário
foram concluídos: veja [BUILD-VALIDATION.md](BUILD-VALIDATION.md). **Nenhum serviço
CoreScope permanente foi instalado ou ativado.** Não altera a lógica Go ou a interface upstream.
Não houve validação em navegador de uma instância CoreScope nesta etapa.

## Memória

| Camada | Valor inicial |
|---|---:|
| packetStore.maxMemoryMB | 64 MiB de orçamento do armazenamento de pacotes |
| Servidor: GOMEMLIMIT | 192 MiB, limite suave do runtime Go |
| Ingestor: GOMEMLIMIT | 96 MiB, limite suave do runtime Go |
| Servidor: MemoryHigh / MemoryMax | 256 / 320 MiB |
| Ingestor: MemoryHigh / MemoryMax | 128 / 192 MiB |
| Slice compartilhada: MemoryHigh / MemoryMax | 384 / **512 MiB** |

Os limites de processo/slice são cgroups systemd, incluindo consumo que o Go não
contabiliza. Os GOMEMLIMIT são suaves, não tetos RSS, e vencem runtime.maxMemoryMB
do JSON. No upstream examinado, o fallback do JSON recebe margem 1,5x; as units
fixam explicitamente os valores efetivos com ambiente. packetStore é orçamento
de dados/indexação estimada, não a memória total do processo. MemoryMax inclui
memória contabilizada pelo kernel e pode encerrar/reiniciar um processo que
exceder o teto. Os limites não reservam essa quantidade de RAM antecipadamente.

A disponibilidade do controlador memory foi conferida no equipamento. A
aplicação efetiva dos limites só poderá ser conferida quando as units forem
instaladas/iniciadas; nesta etapa não criamos um cgroup em execução.
Valores são uma configuração inicial para medir com o tráfego CWB, não uma
alegação de consumo medido ou garantia de que a aplicação caberá nesses limites.

## Menor conjunto de trabalho

- Apenas mqttSources CWB-local, tópico meshcore/CWB/# e iataFilter CWB.
- Credenciais são placeholders CONFIGURE_LOCALLY; preenchê-las só na cópia local.
  Não versionar o config real, senhas ou endereços privados do operador.
- Porta HTTP proposta: 8089, mantendo a página do cadastro existente independente.
- Janela de pacotes em RAM: 6 h; carga inicial quente: 1 h.
- Retenção de pacotes em SQLite: 1 dia; cadastro de nós: 30 dias, observadores: 7 dias.
  Essas são regras para um banco CoreScope novo, não para o cadastro do meshcoreSDR.
- Buffer de ingestão: 1024, em vez dos 50000 do exemplo upstream.
- Carga SQLite em blocos de 500; sem VACUUM completo automático na inicialização.
- GOMAXPROCS=2 por programa. Recursos opcionais de contas, cobertura móvel,
  regiões de cliente e amostras de ambiente RF desligados.
- Região inicial e centro do mapa: CWB / centro público aproximado de Curitiba.
- Horários padrão absolutos em UTC-3: identificador IANA Etc/GMT+3
  (o sinal desse identificador é invertido).

## Preparação para instalação futura

1. Fixar uma revisão/release e instalar toolchain Go compatível em prefixo isolado.
   A base examinada pede Go 1.26 e CGO_ENABLED=1; GCC precisa estar disponível.
   Compilar server e ingestor conforme Makefile, com concorrência baixa na placa.
   **Não usar CGO_ENABLED=0:** SQLite geraria um stub sem funcionalidade.
2. Criar usuário de sistema corescope-cwb. Instalar os executáveis em
   /opt/corescope-cwb/bin e os arquivos estáticos upstream em
   /opt/corescope-cwb/public. Não criar servidor Node.js.
3. Copiar config.example.json para /etc/corescope-cwb/config.json e configurar
   localmente as credenciais do broker. Config real legível pelo usuário de serviço,
   sem publicação no Git. Broker não deve aceitar placeholder literalmente.
4. Instalar slice e duas units em /etc/systemd/system, daemon-reload.
   Ingestor cria StateDirectory/RuntimeDirectory próprios; banco independente em
   /var/lib/corescope-cwb/meshcore.db. Stats transitórios em /run/corescope-cwb/.
5. Iniciar ingestor e servidor; conferir SQLite funcional, MQTT, HTTP e navegador.
   Observar com systemctl show MemoryCurrent/MemoryMax e systemd-cgtop; confirmar
   tetos no cgroup pai e filhos, consumo estável, GC e continuidade dos decoders SDR.
6. Validar API, mapa e atualização ao vivo em navegador antes de considerar implantação
   concluída. Só então habilitar início automático. Não executar manage.sh setup
   para a instalação nativa sem revisar sua automação Docker.

## Validação deste perfil

JSON parseado; conferidos filtro exclusivo CWB, orçamento do store menor que o
limite Go do servidor, soma dos tetos dos serviços igual ao teto da slice,
credenciais-placeholder, porta independente e ausência de mudanças no código Go.
As regras acima também documentam como repetir a conferência manual nas units.
Não foram executadas suites Go/frontend, pois não houve alteração de lógica;
compilação e testes de execução dependem da preparação de toolchain posterior.
Os valores são configuráveis no perfil e nas units; integração ao customizer
não foi alterada, pois os campos de aplicação já existem no upstream e os
limites de cgroup pertencem ao ambiente de implantação.
