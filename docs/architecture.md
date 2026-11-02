# Aphrodite HomeLab — Arquitetura

> Documento de referência da infraestrutura do Aphrodite HomeLab.
>
> **Estado:** arquitetura inicial em funcionamento  
> **Última atualização:** 2026-11-02

## 1. Visão geral

O Aphrodite é um homelab doméstico executado em uma placa Arduino UNO Q com Debian GNU/Linux 13 (Trixie), arquitetura ARM64 (`aarch64`).

A infraestrutura utiliza Docker Compose para executar serviços isolados em uma rede Docker compartilhada. O Caddy atua como proxy reverso HTTP, o AdGuard Home fornece resolução DNS local opcional e o Uptime Kuma monitora a disponibilidade das aplicações.

O acesso é destinado à rede local. Não há, nesta arquitetura, publicação intencional dos serviços na internet.

## 2. Hardware e sistema operacional

| Item | Configuração |
|---|---|
| Dispositivo | Arduino UNO Q |
| Memória RAM | 4 GB |
| Sistema operacional | Debian GNU/Linux 13 (Trixie) |
| Arquitetura | ARM64 (`aarch64`) |
| Hostname | `aphrodite` |
| Interface de rede utilizada | Wi-Fi (`wlan0`) |
| Endereço IPv4 atual | `192.168.15.42` |
| Armazenamento | eMMC de aproximadamente 29 GB |

O endereço IPv4 da placa está reservado no servidor DHCP do roteador. A reserva evita que o endereço mude após uma renovação de concessão, mas não substitui uma configuração de DNS.

## 3. Organização de diretórios

A estrutura principal separa os arquivos de configuração, mantidos no repositório, dos dados persistentes dos serviços:

```text
/home/arduino/
├── aphrodite/
│   ├── docs/
│   ├── infrastructure/
│   │   └── docker/
│   ├── services/
│   │   ├── adguard/
│   │   ├── caddy/
│   │   ├── homepage/
│   │   └── uptime-kuma/
│   └── applications/
│
├── aphrodite-data/
│   ├── adguard/
│   │   ├── conf/
│   │   └── work/
│   ├── caddy/
│   │   ├── config/
│   │   └── data/
│   ├── homepage/
│   └── uptime-kuma/
│
└── docker-data/
```

- `aphrodite/`: repositório do projeto; contém configurações, definições Compose e documentação.
- `aphrodite-data/`: dados persistentes das aplicações. Deve ser preservado ao atualizar ou recriar containers.
- `docker-data/`: diretório interno do Docker Engine. Não deve ser manipulado manualmente.

O Docker Engine utiliza `/home/arduino/docker-data` como `data-root`, configurado em `/etc/docker/daemon.json`.

## 4. Rede e resolução de nomes

### 4.1 Rede Docker

Os serviços participam de uma rede bridge externa chamada `aphrodite`, criada previamente no Docker.

Os containers podem se comunicar por seus nomes de serviço/container, sem precisar utilizar endereços IP internos fixos:

| Serviço | Nome utilizado na rede | Porta interna |
|---|---|---:|
| Caddy | `caddy` | 80 |
| Homepage | `homepage` | 3000 |
| Uptime Kuma | `uptime-kuma` | 3001 |
| AdGuard Home | `adguard` | 80 (HTTP) / 53 (DNS) |

Os endereços IP atribuídos aos containers são dinâmicos e não devem ser utilizados como referências permanentes.

### 4.2 DNS local

O AdGuard Home mantém rewrites DNS para os nomes locais:

| Nome | Endereço |
|---|---|
| `homepage.home.arpa` | `192.168.15.42` |
| `uptime.home.arpa` | `192.168.15.42` |
| `adguard.home.arpa` | `192.168.15.42` |

O domínio `home.arpa` é reservado para uso em redes domésticas.

O roteador continua responsável pelo DHCP e pelo DNS padrão da rede. O AdGuard **não é o DNS obrigatório de toda a rede**: dispositivos que precisarem resolver os nomes `*.home.arpa` devem ser configurados para consultar o AdGuard.

Essa escolha permite desligar a placa sem retirar do ar o serviço DNS padrão da rede doméstica.

### 4.3 Particularidade do Fedora

No computador Fedora utilizado para administrar o homelab, a conexão Wi-Fi foi configurada para ignorar servidores DNS IPv6 recebidos automaticamente do roteador:

```bash
sudo nmcli connection modify "Familia Ramos Vivo-5G" \
  ipv6.ignore-auto-dns yes
```

O objetivo é evitar que o DNS IPv6 do roteador, que não conhece os rewrites locais, interfira na resolução dos nomes `*.home.arpa`.

Essa configuração é específica desse cliente e não altera o funcionamento de IPv6 em geral.

## 5. Serviços

### 5.1 Caddy — proxy reverso

O Caddy recebe as requisições HTTP na porta 80 da placa e as encaminha ao serviço correspondente na rede Docker.

| Endereço local | Destino interno |
|---|---|
| `http://homepage.home.arpa` | `homepage:3000` |
| `http://uptime.home.arpa` | `uptime-kuma:3001` |
| `http://adguard.home.arpa` | `adguard:80` |

Configuração atual do `Caddyfile`:

```caddyfile
http://homepage.home.arpa {
    reverse_proxy homepage:3000
}

http://uptime.home.arpa {
    reverse_proxy uptime-kuma:3001
}

http://adguard.home.arpa {
    reverse_proxy adguard:80
}
```

A configuração utiliza explicitamente `http://`. HTTPS local ainda não faz parte da configuração de acesso atual.

### 5.2 Homepage — dashboard

- **Função:** dashboard para acesso aos serviços do homelab.
- **Container:** `homepage`.
- **Porta interna:** `3000`.
- **Acesso:** `http://homepage.home.arpa`.
- **Persistência:** configuração em `~/aphrodite/services/homepage/config/`.
- **Rede:** `aphrodite`.

O dashboard é publicado pelo Caddy; a porta 3000 não é publicada diretamente no host.

A configuração atual também monta o socket Docker em modo somente leitura para integração com o Docker. Mesmo em modo somente leitura, o acesso ao socket Docker é sensível e deve ser considerado na revisão de segurança.

### 5.3 Uptime Kuma — monitoramento

- **Função:** monitoramento de disponibilidade dos serviços.
- **Container:** `uptime-kuma`.
- **Porta interna:** `3001`.
- **Acesso:** `http://uptime.home.arpa`.
- **Persistência:** `/home/arduino/aphrodite-data/uptime-kuma`.
- **Banco de dados:** SQLite.
- **Rede:** `aphrodite`.

Monitores HTTP já configurados:

| Monitor | URL consultada |
|---|---|
| Homepage | `http://homepage:3000` |
| Uptime Kuma | `http://uptime-kuma:3001` |
| AdGuard Home | `http://adguard:80` |

Os destinos usam nomes internos Docker. Assim, o monitoramento não depende do DNS local nem do proxy reverso.

Os três monitores foram verificados como `UP`. Uma resposta HTTP de redirecionamento (`302`) pode ser válida para os endpoints de Uptime Kuma e AdGuard, que direcionam o navegador para suas telas de autenticação.

Esses monitores verificam a resposta HTTP das interfaces, não todas as funcionalidades dos serviços. Um monitor DNS específico deve ser utilizado para verificar a resolução DNS de fato.

### 5.4 AdGuard Home — DNS local

- **Função:** resolução DNS local e gerenciamento de rewrites.
- **Container:** `adguard`.
- **Porta DNS:** `53` TCP/UDP.
- **Interface web:** porta `80` dentro do container, publicada como `8080` no host.
- **Acesso pela interface web:** `http://adguard.home.arpa`.
- **Acesso direto de administração:** `http://192.168.15.42:8080`.
- **Persistência:**
  - `/home/arduino/aphrodite-data/adguard/work`
  - `/home/arduino/aphrodite-data/adguard/conf`
- **Rede:** `aphrodite`.

O AdGuard fornece os registros locais necessários para acessar os serviços pelo domínio `home.arpa`. Ele não substitui o servidor DHCP nem o DNS padrão do roteador.

## 6. Fluxos de acesso

### 6.1 Acesso de um cliente a uma aplicação

```text
Navegador no cliente
       |
       | 1. Consulta DNS: homepage.home.arpa
       v
AdGuard Home (192.168.15.42:53)
       |
       | 2. Responde: 192.168.15.42
       v
Cliente acessa 192.168.15.42:80
       |
       v
Caddy
       |
       | 3. Encaminha para homepage:3000
       v
Homepage (rede Docker aphrodite)
```

O fluxo é equivalente para Uptime Kuma e AdGuard, alterando o hostname e o destino interno do proxy.

### 6.2 Monitoramento

```text
Uptime Kuma
    |
    +---- HTTP ----> homepage:3000
    |
    +---- HTTP ----> uptime-kuma:3001
    |
    +---- HTTP ----> adguard:80
```

O monitoramento ocorre dentro da rede Docker compartilhada e não passa pelo Caddy.

## 7. Portas publicadas no host

Estado conhecido da configuração:

| Porta no host | Protocolo | Serviço | Finalidade |
|---:|---|---|---|
| 53 | TCP/UDP | AdGuard Home | Consultas DNS |
| 80 | TCP | Caddy | Proxy reverso HTTP |
| 8080 | TCP | AdGuard Home | Interface administrativa direta |

As portas internas `3000` (Homepage) e `3001` (Uptime Kuma) não são publicadas diretamente no host; o acesso normal ocorre pelo Caddy.

A configuração observada do AdGuard também anunciava portas internas adicionais no `docker ps` (como `3000`, `443` e `853`), mas elas não estavam publicadas no host no estado registrado.

## 8. Docker e persistência

O Docker Engine utiliza o seguinte `data-root`:

```json
{
  "data-root": "/home/arduino/docker-data",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "2"
  }
}
```

Os logs dos containers utilizam o driver `json-file`, com rotação limitada a 10 MB por arquivo e dois arquivos por container.

Os dados de aplicações devem permanecer em `aphrodite-data/`, fora do diretório interno gerenciado pelo Docker. Ao recriar um container, os volumes persistentes devem ser preservados.

**Atenção:** não executar comandos destrutivos como `docker system prune -a` sem revisar previamente quais imagens e recursos serão removidos. O ambiente também contém imagens utilizadas pelo Arduino App Lab.

## 9. Segurança: estado atual e pontos de atenção

- Os serviços foram planejados para uso na rede local; não há intenção de expô-los diretamente à internet.
- O Caddy publica a porta HTTP 80, mas ainda não há HTTPS local configurado para o acesso atual.
- A interface administrativa do AdGuard está acessível pela porta 8080 do host e também por meio do Caddy.
- Homepage possui acesso ao socket Docker, ainda que montado como somente leitura. Esse acesso merece revisão.
- As credenciais administrativas devem ser armazenadas de forma segura e não devem ser adicionadas ao repositório.
- O acesso às interfaces administrativas deve ser limitado à rede local e, futuramente, pode ser reforçado com autenticação e regras de firewall.
- Backups dos diretórios persistentes ainda devem ser planejados e testados.

## 10. Procedimentos operacionais

Os comandos abaixo devem ser executados na placa, a partir do diretório do serviço correspondente.

Verificar containers:

```bash
docker ps
```

Verificar logs de um serviço:

```bash
docker logs --tail 100 caddy
docker logs --tail 100 homepage
docker logs --tail 100 uptime-kuma
docker logs --tail 100 adguard
```

Validar os rewrites DNS diretamente no AdGuard:

```bash
dig @192.168.15.42 homepage.home.arpa +short
dig @192.168.15.42 uptime.home.arpa +short
dig @192.168.15.42 adguard.home.arpa +short
```

Validar o encaminhamento do Caddy a partir da placa:

```bash
curl -sS -o /dev/null -w "Homepage: %{http_code}\n" \
  -H "Host: homepage.home.arpa" http://127.0.0.1

curl -sS -o /dev/null -w "Uptime Kuma: %{http_code}\n" \
  -H "Host: uptime.home.arpa" http://127.0.0.1

curl -sS -o /dev/null -w "AdGuard: %{http_code}\n" \
  -H "Host: adguard.home.arpa" http://127.0.0.1
```

Para atualizar ou recriar um serviço, entrar no diretório correspondente e revisar o Compose antes de executar:

```bash
docker compose config
docker compose up -d
```

## 11. Próximas melhorias

Itens sugeridos, ainda não considerados concluídos:

- [ ] Criar monitor DNS no Uptime Kuma para verificar a porta 53 e os rewrites do AdGuard.
- [ ] Criar monitor HTTP para o Caddy.
- [ ] Configurar notificações do Uptime Kuma.
- [ ] Definir e testar uma rotina de backup de `aphrodite-data/`.
- [ ] Documentar o procedimento de restauração.
- [ ] Revisar exposição das interfaces administrativas e regras de firewall.
- [ ] Avaliar HTTPS local e confiança da autoridade certificadora nos clientes.
- [ ] Fixar versões das imagens Docker em vez de depender de tags mutáveis como `latest`.
- [ ] Revisar a necessidade de montar o socket Docker no Homepage.

---

## Histórico de decisões

- O roteador permanece responsável pelo DHCP e pelo DNS padrão da rede.
- O AdGuard é utilizado como DNS opcional para clientes que precisam resolver os nomes locais.
- Os nomes locais usam o domínio reservado `home.arpa`.
- O Caddy centraliza o acesso HTTP às aplicações.
- Os serviços compartilham a rede Docker externa `aphrodite`.
- Os dados persistentes ficam separados do repositório e do `data-root` do Docker.
- O Uptime Kuma utiliza SQLite para evitar a necessidade de um banco de dados separado.
