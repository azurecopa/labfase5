# 🏆 Copa do Mundo Azure — Guia do Evento TFTEC (FIFA 2026 Tickets — cenário VM)

> ⚽ **Bem-vindo(a) ao gramado!** Neste evento você vai **construir do zero** o seu próprio ambiente em nuvem e colocar no ar a aplicação **FIFA 2026 Tickets** — a bilheteria oficial (fictícia) da Copa do Mundo — em **3 Máquinas Virtuais**.
>
> 🥅 **Para todos os níveis.** Você não precisa ser sênior. Cada passo é explicado em detalhe, com o **caminho visual pelo Portal do Azure** sempre que possível — aqui a ideia é **entender o que você está fazendo**.

> 🚧 **Documento vivo.** Itens marcados com _⚠️ a confirmar_ serão fixados conforme o evento se aproxima (ex.: URL do repositório público, dataset do banco). A estrutura, a arquitetura e os passos já valem.

> 🎟️ **Tem um app irmão!** Existe também o **Bolão TFTEC** (palpites). Este guia é o do **Tickets** (venda de ingressos) — arquitetura diferente (banco **relacional** + 3 VMs). Se você for fazer os dois, repare nas diferenças: é parte do aprendizado.

> 🆚 **Existe outro guia deste mesmo app!** O [`GUIA-EVENTO.md`](GUIA-EVENTO.md) cobre o **cenário PaaS** (Web Apps + Azure SQL). Este aqui é o **cenário VM** — você sente na pele o que o PaaS resolve. Faça os dois se quiser entender o trade-off.

---

## 📋 Índice

1. [Sobre a aplicação](#-1-sobre-a-aplicação)
2. [Objetivos do evento](#-2-objetivos-do-evento)
3. [Tecnologias Azure que vamos usar](#-3-tecnologias-azure-que-vamos-usar)
4. [Arquitetura da aplicação](#-4-arquitetura-da-aplicação)
5. [A jornada do aluno](#-5-a-jornada-do-aluno)
   - [🎽 Fase 0 — Pré-jogo: pré-requisitos](#-fase-0--pré-jogo-pré-requisitos)
   - [🤝 Fase 1 — Convocação: fork do repositório](#-fase-1--convocação-fork-do-repositório)
   - [🏟️ Fase 2 — Fase de Grupos: provisionar a VNet e as 3 VMs](#️-fase-2--fase-de-grupos-provisionar-a-vnet-e-as-3-vms)
   - [🗄️ Fase 3 — Oitavas: configurar a `vm-db` (SQL Server + bacpac)](#️-fase-3--oitavas-configurar-a-vm-db-sql-server--bacpac)
   - [🔧 Fase 4 — Quartas: configurar a `vm-back` (backend Node)](#-fase-4--quartas-configurar-a-vm-back-backend-node)
   - [⚙️ Fase 5 — Semifinal: configurar a `vm-front` (frontend + proxy reverso)](#️-fase-5--semifinal-configurar-a-vm-front-frontend--proxy-reverso)
   - [🏆 Fase 6 — Final: smoke test e comemorar](#-fase-6--final-smoke-test-e-comemorar)
   - [🎖️ Fase 7 — Pós-jogo: troubleshooting + desligar as VMs](#️-fase-7--pós-jogo-troubleshooting--desligar-as-vms)
6. [Tabela de variáveis e segredos](#-6-tabela-de-variáveis-e-segredos)
7. [Evolução de segurança (o "VAR" da arquitetura)](#️-7-evolução-de-segurança-o-var-da-arquitetura)

---

## ⚽ 1. Sobre a aplicação

O **FIFA 2026 Tickets** é a **bilheteria** (fictícia, educacional) da Copa do Mundo 2026: o torcedor navega pelos jogos, estádios e seleções, escolhe um setor e **compra ingressos**, recebendo um **ingresso premium com QR code** validável.

É uma aplicação **3 camadas clássica** — o pão-com-manteiga da web corporativa:

- 🎟️ **Catálogo** de jogos (104 partidas), 17 estádios oficiais e 49 seleções
- 🛒 **Fluxo de compra** com ocupação por jogo e bloqueio de esgotado
- 🪪 **Ingresso premium** com QR code real + página de validação
- 📊 **Painel admin** (vendas, usuários, resultados) com paginação server-side
- 🔐 **Autenticação própria** (JWT) + papel admin
- 📚 Conteúdo: História das Copas, quiz, bracket do mata-mata

> 💡 **Por que esse app neste cenário VM?** Ele ensina **operação real de infraestrutura**: você instala IIS, Node.js, iisnode e SQL Server **com as suas próprias mãos**, configura NSG, monta um padrão **jump host**, e depois vê **na prática** o que cada serviço PaaS resolve. É o cenário que TI corporativa **ainda mantém** em muitos lugares — saber operar isso é diferencial.

---

## 🎯 2. Objetivos do evento

Ao final, você terá feito **com as suas próprias mãos**:

| # | Você vai aprender a... |
|---|---|
| 1 | Criar **VNet + subnets + NSGs** no Portal e desenhar uma rede 3-camadas com isolamento |
| 2 | Provisionar **3 Máquinas Virtuais Windows Server** (uma pública, duas privadas) |
| 3 | Usar **RDP** + padrão **jump host** para administrar VMs privadas sem expor à Internet |
| 4 | Instalar e configurar **SQL Server 2022** numa VM e **restaurar um banco via `.bacpac`** |
| 5 | Instalar e configurar **IIS + iisnode + Node** para hospedar uma API Express no Windows |
| 6 | Instalar **URL Rewrite + ARR** para fazer **proxy reverso** `/api/*` → backend privado |
| 7 | Validar a aplicação ponta a ponta + confirmar que back/db **não respondem à Internet** |

> 🧠 **Filosofia:** **Portal-first** (clicar e ver). PowerShell e `az cli` apenas **dentro da VM via RDP** ou para desligar tudo no final. Você sai sabendo **o que cada peça faz e por quê**.

> 🆚 **Diferença para o cenário PaaS (`GUIA-EVENTO.md`):** lá você cria 2 Web Apps + Azure SQL e termina em 1h30 com o app no ar. Aqui você gasta ~45 min só para subir as VMs, mais 1h instalando software — e termina com a **mesma aplicação** rodando. **Esse contraste é o ponto.**

---

## ☁️ 3. Tecnologias Azure que vamos usar

Tudo dentro de **um Resource Group** (`fifa2026-vm-rg`). Nomes de VMs são **únicos por RG** (você pode usar os mesmos que estão aqui).

| Serviço Azure | Para que serve no Tickets | Camada / Custo |
|---|---|---|
| 🌐 **Virtual Network** `vnet-fifa2026` | Rede privada onde as 3 VMs se enxergam pelos IPs internos | Grátis |
| 🏷️ **Network Security Groups (3)** | Firewalls por subnet: front 80/443, back 3001, db 1433 | Grátis |
| 🖥️ **Virtual Machine** `vm-front` (Windows Server 2022, B2s, **IP público**) | Hospeda IIS+ARR — único ponto de entrada da Internet | B2s ~$30/mês (24/7) |
| 🖥️ **Virtual Machine** `vm-back` (Windows Server 2022, B2s, **sem IP público**) | Hospeda IIS+iisnode+Node — backend da API | B2s ~$30/mês (24/7) |
| 🖥️ **Virtual Machine** `vm-db` (Windows Server 2022, B2s, **sem IP público**) | Hospeda SQL Server 2022 Developer (gratuito) com o banco | B2s ~$30/mês (24/7) |
| 💾 **Managed Disks** (3 × 127 GB) | Disco do OS de cada VM (incluído na VM, ~$5/mês cada se VM desligada) | inclusos |

> 💰 **Custo total real:** ~**$90/mês** se as 3 VMs ficarem **24/7**. Mas você **NÃO precisa deixar ligado** — use `az vm deallocate` ao final de cada sessão (passo na Fase 7) e o compute para de cobrar (paga só ~$5/mês por disco). Configure um **alerta de orçamento** (Fase 0).

> 🔐 **Sobre segredos:** este cenário VM usa **arquivos `.env` nos próprios servidores** (vm-back) + **senha do SQL anotada por você**. Modelo simples, mas frágil — qualquer um com acesso à VM lê o `.env`. _Boa prática de produção:_ migrar segredos para um **Azure Key Vault** com Managed Identity — fora do escopo deste guia (você vai vê-lo no cenário PaaS evoluído).

---

## 🗺️ 4. Arquitetura da aplicação

O "mapa do estádio" — **Cenário 3 VMs** (o que você vai montar):

```
                          🌎 TORCEDOR (navegador / celular)
                                      │  HTTP/HTTPS
                                      ▼
        ┌─────────────────────────────────────────────────────────┐
        │   🌐 vm-front  (Windows Server, IP público)             │
        │   • IIS :80  servindo dist/ do React                    │
        │   • URL Rewrite + ARR (proxy)                           │
        │   • web.config: reescreve  /api/*  ─────────────────────┼──┐
        └─────────────────────────────────────────────────────────┘  │
                                                                      │  /api/*  (IP privado)
                                                                      ▼
        ┌─────────────────────────────────────────────────────────┐
        │   🔒 vm-back  (Windows Server, SEM IP público)          │
        │   • IIS + iisnode :3001                                 │
        │   • Node 18 LTS executa fifa2026-api/                   │
        │   • .env aponta para IP privado da vm-db                │
        └──────────────────────────────┬──────────────────────────┘
                                        │ driver mssql (porta 1433, IP privado)
                                        ▼
        ┌─────────────────────────────────────────────────────────┐
        │   🟥 vm-db  (Windows Server, SEM IP público)            │
        │   • SQL Server 2022 Developer (gratuito)                │
        │   • Database: FIFA2026Tickets (restaurado do bacpac)    │
        └─────────────────────────────────────────────────────────┘

        ──────────────────────────────────────────────────────────
        🧗 PADRÃO JUMP HOST (para administrar back/db):
            seu computador  ──RDP IP público──▶  vm-front
                                                    │
                                                    └─RDP IP privado──▶  vm-back / vm-db
        ──────────────────────────────────────────────────────────
```

**Princípios de design (e o que isso ensina):**

- 🧅 **Defesa em profundidade.** Só `vm-front` é pública (80/443/3389). `vm-back` aceita 3001 apenas da subnet do front; `vm-db` aceita 1433 apenas da subnet do back. **3 camadas, cada uma protegendo a próxima.**
- 🔁 **Sem CORS em produção.** O navegador chama `/api/*` na **mesma origem** do frontend; o `web.config` faz o *reverse proxy* via ARR para o backend (`http://<IP_BACK>:3001/api/...`). Você aprende o padrão *proxy reverso* na sua versão "raiz" — direto no IIS.
- 🗃️ **Migração de dados real.** O banco não nasce vazio: você **importa um `.bacpac`** via SSMS na `vm-db` — exatamente como se faz ao mover um sistema legado para a nuvem.
- 🧗 **Jump host.** As VMs privadas (`vm-back`, `vm-db`) **não recebem IP público**. Você as administra entrando primeiro na `vm-front` por RDP, e de lá saltando para o IP privado das outras. É como o pessoal de SRE faz **há décadas**.

> 🆚 **No cenário PaaS (`GUIA-EVENTO.md`)** todos esses controles existem também, mas o Azure os entrega prontos: Web App tem TLS embutido, Access Restriction substitui NSG, Azure SQL substitui SQL Server. Aqui você vê a **versão manual** — depois entende por que PaaS economiza tempo.

### 🛡️ NSG Matrix

| NSG | Inbound permitido | Inbound negado |
|---|---|---|
| `nsg-front` | TCP 80, 443 (Internet); TCP 3389 (apenas o **seu IP**) | Resto |
| `nsg-back`  | TCP 3001 (origem: subnet do front); TCP 3389 (origem: subnet do front) | Internet |
| `nsg-db`    | TCP 1433 (origem: subnet do back); TCP 3389 (origem: subnet do front) | Internet, subnet do front (exceto 3389) |

---

## 🧭 5. A jornada do aluno

| Fase | Etapa | Tempo aprox. |
|---|---|---|
| 🎽 Pré-jogo | 0. Pré-requisitos | 10 min |
| 🤝 Convocação | 1. Fork do repositório + baixar bacpac | 5 min |
| 🏟️ Fase de Grupos | 2. Provisionar VNet + 3 VMs no Portal | 25 min |
| 🗄️ Oitavas | 3. SQL Server na `vm-db` + restore bacpac | 20 min |
| 🔧 Quartas | 4. IIS+iisnode+Node na `vm-back` + deploy backend | 20 min |
| ⚙️ Semifinal | 5. IIS+ARR na `vm-front` + build+deploy frontend | 15 min |
| 🏆 Final | 6. Smoke test ponta a ponta | 10 min |
| 🎖️ Pós-jogo | 7. Troubleshooting + **desligar VMs** | livre |

> 🧩 **Como o código chega até você:** baixe o ZIP da aplicação direto na VM (link na Fase 4). Sem CI/CD nesse cenário — _esse_ é o caminho PaaS, e parte da lição é sentir a falta dele.

> 🧠 **Total esperado:** ~1h45 de mão na massa + tempo de download/instalação. Reserve **2h30 cheias** na primeira execução.

---

### 🎽 Fase 0 — Pré-jogo: pré-requisitos

- [ ] **Conta Azure ativa** — [azure.microsoft.com/free](https://azure.microsoft.com/free/)
- [ ] **Cliente RDP**:
  - **Windows:** já vem com `mstsc` (Conexão de Área de Trabalho Remota) — busque "Remote Desktop Connection" no menu Iniciar
  - **macOS:** instale **Microsoft Remote Desktop** na Mac App Store
  - **Linux:** instale **Remmina** (`sudo apt install remmina remmina-plugin-rdp`)
- [ ] **Navegador moderno** (para o Portal e para testar o app no fim)
- [ ] **Bloco de notas** para anotar: IPs (público da `vm-front`, privados das 3 VMs), senha de admin do SQL, senha de admin do Windows Server, JWT_SECRET

**Confirme o Azure:** entre em [portal.azure.com](https://portal.azure.com) → topo direito → **Subscription** ativa.

**Alerta de orçamento (essencial neste cenário):** Portal → **Cost Management → Budgets → + Add** → **$30/mês**, alerta em 80% e 100% → seu e-mail.

> ⚠️ **Por que o alerta é crítico aqui?** No cenário PaaS, esquecer ligado custa ~$18/mês. Aqui custa **~$90/mês** se as 3 VMs ficarem 24/7. **Sempre desligue ao final da sessão** (Fase 7).

> ✅ **Pronto quando:** você abre o Portal, vê uma subscription ativa, e seu cliente RDP abre sem erro.

---

### 🤝 Fase 1 — Convocação: fork do repositório

1. Acesse o repositório público: **`https://github.com/TFTEC/<repo-publico-tickets>`** _(⚠️ a confirmar — URL final no evento)_
2. **Fork** (canto superior direito) → **Create fork**
3. No SEU fork, encontre e baixe o arquivo `FIFA2026-APP/FIFA2026Tickets.bacpac` (botão **Download**) — **guarde no seu computador local**. Você vai subir esse arquivo para a `vm-db` na Fase 3.

> ⚠️ **Atenção ao bacpac:** use **sempre** a versão do `.bacpac` que está no seu fork (atualizado pela organização antes do evento). Se você usar um `.bacpac` antigo, o app vai subir com dados desatualizados (12 jogos em vez de 104).

> ✅ **Pronto quando:** existe um fork seu + você tem o `FIFA2026Tickets.bacpac` baixado localmente.

---

### 🏟️ Fase 2 — Fase de Grupos: provisionar a VNet e as 3 VMs

Tudo em [portal.azure.com](https://portal.azure.com). Use a **barra de busca** no topo, abra o serviço, **+ Create**, e finalize em **Review + create → Create**.

#### 🎽 Passo 0 — Resource Group e Região

1. Busca → **Resource groups** → **+ Create**
2. **Subscription:** a sua · **Resource group:** `fifa2026-vm-rg` · **Region:** **East US**
3. **Review + create** → **Create**

📋 **Anote:** mantenha **East US** para todos os recursos a seguir.

---

#### 1️⃣ Virtual Network — `vnet-fifa2026`

1. Busca → **Virtual networks** → **+ Create**
2. **Resource group:** `fifa2026-vm-rg`
3. **Name:** `vnet-fifa2026`
4. **Region:** East US
5. Aba **IP addresses** → **IPv4 address space:** `10.20.0.0/16`
6. Subnets: deixe a `default` (`10.20.0.0/24`) — vamos usar essa para as 3 VMs
7. **Review + create** → **Create**

> 💡 **Por que uma subnet só?** Para o nível introdutório, simplifica. Em produção real você teria subnets separadas (web/app/data) com NSGs distintas — o princípio é o mesmo.

---

#### 2️⃣ vm-front — VM pública (porta de entrada)

1. Busca → **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Resource group:** `fifa2026-vm-rg`
3. **Virtual machine name:** `vm-front`
4. **Region:** East US
5. **Availability options:** No infrastructure redundancy required (workshop)
6. **Image:** **Windows Server 2022 Datacenter: Azure Edition** (x64, Gen2)
7. **Size:** **Standard_B2s** (2 vCPU, 4 GB RAM — `Change size` se precisar)
8. **Administrator account → Username:** `tftecadmin` · **Password:** crie uma forte e 📋 **anote** (rótulo: *VM Admin Password*)
9. **Public inbound ports:** **Allow selected ports** → marque **RDP (3389)** e **HTTP (80)** e **HTTPS (443)**
10. Aba **Networking:** **Virtual network:** `vnet-fifa2026` · **Subnet:** `default` · **Public IP:** _(criar novo, padrão)_ · **NIC NSG:** **Advanced** · _(deixe que a wizard crie um NSG para a VM)_
11. **Review + create** → **Create** (espera 1-2 min)

📋 **Após criar:** abra `vm-front` → seção **Overview** → 📋 **anote** o **Public IP address** (rótulo: *IP_FRONT*) e o **Private IP address** (rótulo: *IP_FRONT_PRIVADO*).

---

#### 3️⃣ vm-back — VM privada (backend Node)

1. Busca → **Virtual machines** → **+ Create** → **Azure virtual machine**
2. **Resource group:** `fifa2026-vm-rg`
3. **Virtual machine name:** `vm-back`
4. **Region:** East US · **Image:** Windows Server 2022 Datacenter: Azure Edition · **Size:** Standard_B2s
5. **Administrator:** mesmo `tftecadmin` + mesma senha (anotada)
6. **Public inbound ports:** **None** ← (essa VM **não** recebe acesso direto)
7. Aba **Networking:** **Virtual network:** `vnet-fifa2026` · **Subnet:** `default` · **Public IP:** **None** · **NIC NSG:** **Basic** → **None** (vamos configurar depois)
8. **Review + create** → **Create**

📋 **Após criar:** abra `vm-back` → **Overview** → 📋 **anote** o **Private IP address** (rótulo: *IP_BACK*).

---

#### 4️⃣ vm-db — VM privada (banco)

Repita exatamente o mesmo processo da `vm-back`, mudando só:
- **Virtual machine name:** `vm-db`
- (Tudo o mais idêntico — sem IP público, mesma VNet/subnet, mesmo admin)

📋 **Após criar:** abra `vm-db` → **Overview** → 📋 **anote** o **Private IP address** (rótulo: *IP_DB*).

---

#### 5️⃣ Ajustar NSGs (regras de firewall)

A wizard criou um NSG default para a `vm-front`. Vamos refinar e adicionar NSGs específicas para `vm-back` e `vm-db`.

**5a. NSG da `vm-back`:**
1. Busca → **Network security groups** → **+ Create**
2. **RG:** `fifa2026-vm-rg` · **Name:** `nsg-back` · **Region:** East US → **Create**
3. Abra `nsg-back` → **Inbound security rules** → **+ Add**:
   - **Source:** `IP Addresses` → **Source IP:** `10.20.0.0/24` (toda a subnet) · **Destination port ranges:** `3001` · **Protocol:** TCP · **Action:** Allow · **Priority:** 100 · **Name:** `allow-back-from-subnet`
   - **+ Add** outra regra: **Source:** `IP Addresses` → `10.20.0.0/24` · **Destination port:** `3389` · **Action:** Allow · **Priority:** 110 · **Name:** `allow-rdp-from-subnet`
4. **Associate** essa NSG à NIC da `vm-back`: dentro do NSG → **Network interfaces** → **+ Associate** → escolha a NIC da `vm-back`.

**5b. NSG da `vm-db`:**
1. Repita o processo: **+ Create** → `nsg-db`
2. Regras inbound:
   - `allow-sql-from-subnet`: Source `10.20.0.0/24`, port `1433`, Allow, Priority 100
   - `allow-rdp-from-subnet`: Source `10.20.0.0/24`, port `3389`, Allow, Priority 110
3. **Associate** à NIC da `vm-db`.

> 💡 **Por que `10.20.0.0/24` em vez do IP exato da vm-front?** Source `subnet inteira` cobre o caso de quem cria mais VMs depois sem ter que editar NSG. Para produção, prefira **Application Security Groups** (ASG) — mas isso fica para outro dia.

#### ✅ Checklist da Fase 2

No `fifa2026-vm-rg` você deve ver:

```
fifa2026-vm-rg
├── vnet-fifa2026               (VNet 10.20.0.0/16)
├── vm-front                    (Windows, B2s, IP público) + NIC + Disk + NSG (auto)
├── vm-back                     (Windows, B2s, sem IP público) + NIC + Disk
├── vm-db                       (Windows, B2s, sem IP público) + NIC + Disk
├── nsg-back                    (associada à NIC da vm-back)
└── nsg-db                      (associada à NIC da vm-db)
```

Anotado no bloco: *VM Admin Password*, *IP_FRONT*, *IP_FRONT_PRIVADO*, *IP_BACK*, *IP_DB*.

> ✅ **Pronto quando:** as 3 VMs aparecem como **Running** + as NSGs estão associadas.

---

### 🗄️ Fase 3 — Oitavas: configurar a `vm-db` (SQL Server + bacpac)

#### 3.1 Conectar via RDP (padrão jump host)

1. Abra seu cliente RDP. **Conecte primeiro à `vm-front`** usando `IP_FRONT`, user `tftecadmin`, senha anotada.
2. **Dentro da `vm-front`**, abra o RDP do Windows (Iniciar → "Remote Desktop Connection"), e conecte ao `IP_DB` (privado).
3. Login `tftecadmin` + mesma senha.

> 💡 **Por que esse pulo?** A `vm-db` não tem IP público — o único caminho é entrar via outra VM da mesma VNet (jump host). Esse é o padrão padrão de SRE.

#### 3.2 Instalar SQL Server 2022 Developer (gratuito)

Já dentro da `vm-db`:

1. No navegador da VM: [microsoft.com/sql-server/sql-server-downloads](https://www.microsoft.com/sql-server/sql-server-downloads) → baixe **Developer Edition**
2. Execute o instalador → **Basic** → aceite → **Install**
3. Quando terminar, clique em **Customize** → no setup, escolha **Mixed Mode (SQL Server and Windows Authentication)** → defina uma **senha do `sa`** forte → 📋 **anote** (rótulo: *SQL SA Password*) → continue até concluir.

#### 3.3 Habilitar TCP/IP

1. Busca no Windows → **SQL Server 2022 Configuration Manager** → abra
2. **SQL Server Network Configuration → Protocols for MSSQLSERVER** → **TCP/IP** → botão direito → **Enable**
3. Confirme a janela de aviso
4. **SQL Server Services** → **SQL Server (MSSQLSERVER)** → botão direito → **Restart**

#### 3.4 Liberar porta 1433 no firewall do Windows

PowerShell **como Administrador** na `vm-db`:

```powershell
New-NetFirewallRule -DisplayName "SQL Server 1433" -Direction Inbound `
  -Protocol TCP -LocalPort 1433 -Action Allow
```

#### 3.5 Restaurar o `.bacpac`

1. **Baixar SSMS** (SQL Server Management Studio) na `vm-db`: [aka.ms/ssmsfullsetup](https://aka.ms/ssmsfullsetup) → instalar (5-10 min)
2. Abra **SSMS** → Connect: **Server name:** `localhost` · **Authentication:** SQL Server · **Login:** `sa` · **Password:** _SQL SA Password_ → **Connect**
3. **Object Explorer** → botão direito em **Databases** → **Import Data-tier Application...**
4. **Next** → **Browse** → selecione o `FIFA2026Tickets.bacpac` (você precisa copiá-lo do seu computador para dentro da `vm-db` — use copy-paste no RDP, ou suba via Storage)
5. **Database name:** `FIFA2026Tickets` → **Next** → **Next** → **Finish**
6. Aguarde ~1-3 min (depende do tamanho)

#### 3.6 Criar login da aplicação

Ainda no SSMS, **New Query**:

```sql
-- Login (server-level)
CREATE LOGIN fifa2026_db WITH PASSWORD = 'F1f@2026App!';
GO

-- Mapear para o banco
USE FIFA2026Tickets;
CREATE USER fifa2026_db FOR LOGIN fifa2026_db;
ALTER ROLE db_datareader ADD MEMBER fifa2026_db;
ALTER ROLE db_datawriter ADD MEMBER fifa2026_db;
GO
```

📋 **Anote:** *DB_USER* = `fifa2026_db` · *DB_PASSWORD* = a senha acima (troque por uma sua mais forte se preferir).

> ✅ **Pronto quando:** no SSMS, `SELECT COUNT(*) FROM matches;` retorna **104** (e `SELECT COUNT(*) FROM stadiums;` retorna **17**, `SELECT COUNT(*) FROM teams;` retorna **49**).

---

### 🔧 Fase 4 — Quartas: configurar a `vm-back` (backend Node)

#### 4.1 Conectar via jump host

Mesma técnica da Fase 3: RDP na `vm-front` → de lá, RDP no `IP_BACK`.

#### 4.2 Instalar IIS + recursos

PowerShell **como Administrador** na `vm-back`:

```powershell
# Instalar IIS
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Recursos adicionais (Story 1.1 + DEPLOY_IIS_SIMPLIFICADO.md)
Install-WindowsFeature -Name Web-WebSockets
Install-WindowsFeature -Name Web-Stat-Compression
Install-WindowsFeature -Name Web-Dyn-Compression

Write-Host "OK IIS instalado" -ForegroundColor Green
```

#### 4.3 Instalar Node.js e iisnode

Ainda no navegador da `vm-back`:

1. **Node.js LTS** (18 ou 20): [nodejs.org](https://nodejs.org/en/download) → baixe e instale (Windows Installer x64)
2. **iisnode** (versão **Full**): [github.com/Azure/iisnode/releases](https://github.com/Azure/iisnode/releases) → baixe `iisnode-full-v0.2.26-x64.msi` → instale
3. **URL Rewrite Module**: [iis.net/downloads/microsoft/url-rewrite](https://www.iis.net/downloads/microsoft/url-rewrite) → instale

#### 4.4 Baixar a aplicação

Baixe o ZIP da aplicação direto na `vm-back`:

```
https://stoposgraduacaotftec.blob.core.windows.net/arquivos-pos/FIFA2026-APP.zip
```

Extraia para `C:\inetpub\wwwroot\` — vai aparecer a pasta `fifa2026-api/`.

#### 4.5 Configurar `.env` da API

Em `C:\inetpub\wwwroot\fifa2026-api\` crie um arquivo `.env` (use Bloco de Notas → Salvar como, marque "All files" para não virar `.env.txt`):

```env
# Cenário VM
DB_SERVER=<IP_DB>
DB_PORT=1433
DB_USER=fifa2026_db
DB_PASSWORD=F1f@2026App!
DB_NAME=FIFA2026Tickets

PORT=3001
HOST=0.0.0.0

JWT_SECRET=<string longa aleatória>
JWT_EXPIRES_IN=7d

FRONTEND_URL=http://<IP_FRONT>
```

Substitua `<IP_DB>` (anotado na Fase 2), `<IP_FRONT>` (Public IP da vm-front, também anotado), e gere um `JWT_SECRET` longo (qualquer string aleatória de 32+ chars).

> 💡 **Por que `HOST=0.0.0.0`?** Sem isso, Node escuta só em `localhost`, e a vm-front não consegue alcançar pelo IP privado. Com `0.0.0.0` aceita conexões de toda a VNet.

#### 4.6 Instalar dependências do Node

PowerShell **como Administrador** em `C:\inetpub\wwwroot\fifa2026-api`:

```powershell
cd C:\inetpub\wwwroot\fifa2026-api
npm install --omit=dev
```

(~3-5 min)

#### 4.7 Permissões na pasta

```powershell
$acl = Get-Acl "C:\inetpub\wwwroot\fifa2026-api"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("IIS_IUSRS", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl "C:\inetpub\wwwroot\fifa2026-api" $acl
```

#### 4.8 Criar site no IIS (porta 3001)

1. Busca → **IIS Manager** → abra
2. Painel esquerdo → expanda o servidor → botão direito em **Sites** → **Add Website**
3. Preencha:
   - **Site name:** `FIFA2026-API`
   - **Physical path:** `C:\inetpub\wwwroot\fifa2026-api`
   - **Binding type:** http · **IP address:** All Unassigned · **Port:** `3001` · **Host name:** _(vazio)_
4. **OK**
5. **Application Pools** → encontre `FIFA2026-API` → botão direito → **Advanced Settings** → **.NET CLR Version:** `No Managed Code` → **OK**

#### 4.9 Smoke local na `vm-back`

PowerShell:

```powershell
Invoke-RestMethod -Uri "http://localhost:3001/api/health"
Invoke-RestMethod -Uri "http://localhost:3001/api/matches" | Select-Object -ExpandProperty matches | Measure-Object | Select-Object Count
# Deve retornar Count = 104
```

> ✅ **Pronto quando:** `/api/health` responde OK e `/api/matches` retorna 104.

---

### ⚙️ Fase 5 — Semifinal: configurar a `vm-front` (frontend + proxy reverso)

#### 5.1 Build do frontend (no SEU computador local, não na VM)

1. Clone o seu fork localmente (se ainda não tiver):
   ```bash
   git clone https://github.com/<seu-usuario>/<seu-fork>
   cd <seu-fork>/FIFA2026-APP/Lovable/World\ Cup\ Tickets\ Hub
   ```
2. Instale dependências e faça o build apontando para o IP privado da vm-back:
   ```bash
   npm install
   BACKEND_URL=http://<IP_BACK>:3001 npm run build
   ```
   _(No Windows local, use `set BACKEND_URL=...` antes do `npm run build`.)_
3. A pasta `dist/` foi gerada com `web.config` já apontando para o backend correto. **Comprime essa pasta `dist/` em um ZIP** — você vai copiar para a `vm-front`.

#### 5.2 Conectar via RDP direto

A `vm-front` tem IP público, então RDP direto: cliente RDP → `IP_FRONT` → `tftecadmin`.

#### 5.3 Instalar IIS + URL Rewrite + ARR

PowerShell **como Administrador** na `vm-front`:

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-WebSockets
Install-WindowsFeature -Name Web-Stat-Compression
Install-WindowsFeature -Name Web-Dyn-Compression
```

No navegador da `vm-front`:
- [URL Rewrite Module](https://www.iis.net/downloads/microsoft/url-rewrite) → instale
- [Application Request Routing (ARR)](https://www.iis.net/downloads/microsoft/application-request-routing) → instale

#### 5.4 Habilitar proxy no ARR

1. **IIS Manager** → clique no nome do servidor (raiz da árvore)
2. Painel central → duplo clique em **Application Request Routing Cache**
3. Painel direito → **Server Proxy Settings...**
4. ✅ Marque **Enable proxy** → **Apply**

> ⚠️ **Não pule este passo.** Sem `Enable proxy`, o `web.config` do frontend tenta fazer rewrite e o IIS retorna 502. Esse é o erro nº 1 do cenário VM.

#### 5.5 Deploy do `dist/` na `vm-front`

1. Copie o ZIP do `dist/` do seu computador para a `vm-front` (copy-paste no RDP funciona; ou use OneDrive temporariamente)
2. Extraia para `C:\inetpub\wwwroot\fifa2026-web\` — você deve ver `index.html` + `assets/` + `web.config` lá dentro
3. Abra `C:\inetpub\wwwroot\fifa2026-web\web.config` no Bloco de Notas e **confirme** que tem uma regra com `<action type="Rewrite" url="http://<IP_BACK>:3001/api/{R:1}" />` (com o seu IP correto, embutido pelo build)

#### 5.6 Criar site no IIS (porta 80)

1. **IIS Manager** → botão direito em **Sites** → **Add Website**
2. **Site name:** `FIFA2026-Web` · **Physical path:** `C:\inetpub\wwwroot\fifa2026-web` · **Binding:** http, port `80`
3. **OK**
4. **Application Pools** → `FIFA2026-Web` → **Advanced Settings** → `.NET CLR Version` = `No Managed Code`

> ⚠️ **Conflito de porta:** o site **Default Web Site** que vem com o IIS já ocupa a porta 80. Pare ele: **Sites → Default Web Site → Stop** (painel direito), ou delete.

> ✅ **Pronto quando:** abrindo o navegador da `vm-front` em `http://localhost`, o site FIFA 2026 carrega.

---

### 🏆 Fase 6 — Final: smoke test e comemorar

Saia do RDP — agora teste do **seu computador**, com a Internet real.

#### 6.1 No navegador

Abra: **`http://<IP_FRONT>`** (o IP público da `vm-front`)

- [ ] A **home** carrega (lista de jogos/estádios visível)
- [ ] Clique em "**Entrar**" → faça login com `admin@fifa2026.com` / `admin123` (senha padrão do seed do bacpac)
- [ ] Listagem de **jogos** mostra 104 partidas
- [ ] **Cadastre um usuário novo** → faça **login**
- [ ] **Compre um ingresso** → recebe o ingresso premium com **QR code**
- [ ] Acesse a **página de validação** do ingresso (link no QR) → mostra "válido"
- [ ] Login admin → **painel de vendas/usuários** abre

#### 6.2 PowerShell — validação automatizada (no seu computador)

```powershell
$FRONT_IP = "<IP_FRONT>"

# Frontend responde
Invoke-WebRequest "http://$FRONT_IP" -UseBasicParsing | Select-Object StatusCode

# Proxy /api funciona
Invoke-RestMethod "http://$FRONT_IP/api/health"

# Login
$body = @{ email='admin@fifa2026.com'; password='admin123' } | ConvertTo-Json
$r = Invoke-RestMethod "http://$FRONT_IP/api/auth/login" `
  -Method POST -ContentType 'application/json' -Body $body
$r.user.email   # admin@fifa2026.com

# 104 jogos
$h = @{ Authorization = "Bearer $($r.token)" }
(Invoke-RestMethod "http://$FRONT_IP/api/matches" -Headers $h).matches.Count   # 104
```

#### 6.3 Confirme o isolamento (back e db NÃO devem responder à Internet)

Do seu computador, tente alcançar diretamente os IPs **privados** da vm-back:

```powershell
$BACK_IP = "<IP_BACK>"
try {
  Invoke-WebRequest "http://${BACK_IP}:3001/api/health" -TimeoutSec 5 -UseBasicParsing
  Write-Error "FALHOU: backend está acessível pela Internet!"
} catch {
  Write-Host "OK: backend é privado (timeout esperado)" -ForegroundColor Green
}
```

> 🏆 **Conseguiu?** Você publicou uma aplicação 3 camadas em VMs Windows com IIS, iisnode, SQL Server, proxy reverso e isolamento de rede — **a forma "raiz" de fazer**. **É campeão!** 🎉

---

### 🎖️ Fase 7 — Pós-jogo: troubleshooting + desligar as VMs

#### 7.1 Tabela de troubleshooting

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| Front abre, mas `/api/*` retorna 502 | ARR proxy não habilitado | IIS Manager → ARR → Server Proxy Settings → ✅ **Enable proxy** |
| Front abre, mas `/api/*` retorna 404 | URL Rewrite não instalado, ou `web.config` errado | Reinstale URL Rewrite; confira regra `Rewrite` no `dist\web.config` |
| Backend retorna 500 Internal Server Error | `.env` errado, ou `npm install` faltou | Veja `C:\inetpub\wwwroot\fifa2026-api\iisnode\*.log`; rode `npm install --omit=dev` de novo |
| Backend não conecta no SQL | TCP/IP do SQL Server desligado, ou firewall Windows bloqueando 1433 | Configuration Manager → habilite TCP/IP → restart serviço; rode o `New-NetFirewallRule` da Fase 3.4 |
| Backend vê o SQL mas dá "Login failed" | Senha errada no `.env`, ou login `fifa2026_db` não existe | Rode o `CREATE LOGIN` da Fase 3.6; confirme `DB_USER`/`DB_PASSWORD` no `.env` |
| Browser não abre o IP público | NSG da `vm-front` não tem inbound 80; ou Windows Firewall dentro da VM bloqueando | Portal: NSG da `vm-front` → garanta inbound 80/443 da Internet; RDP na vm-front: `New-NetFirewallRule -DisplayName "HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow` |
| RDP na `vm-back` ou `vm-db` falha | NSG sem 3389 da subnet, ou você está tentando do seu IP em vez de via vm-front (jump host) | Confirme NSG; sempre conecte vm-front primeiro, depois vm-back/vm-db pelo IP privado |
| Import do bacpac falha | Versão do SSMS antiga; ou arquivo corrompido na cópia RDP | Baixe SSMS mais recente; refaça a cópia do `.bacpac` |

> 📚 **Documentação adicional:** [`Lovable/World Cup Tickets Hub/DEPLOY_AZURE.md`](../Lovable/World%20Cup%20Tickets%20Hub/DEPLOY_AZURE.md) tem **8 partes detalhadas** se algum passo travar — use como fallback.

#### 7.2 💰 DESLIGAR as VMs (custo zero quando não usadas)

**Esse é o passo mais importante da sessão.** Esqueceu ligado = ~$3/dia.

**Pelo Portal (recomendado):**
1. Busca → **Virtual machines**
2. Para cada VM (`vm-front`, `vm-back`, `vm-db`):
   - Abra → **Stop** (não "Restart"!) → confirme
3. ✅ Status muda para **Stopped (deallocated)** — **compute para de cobrar**

**Pelo Azure Cloud Shell (mais rápido):**
```bash
az vm deallocate -g fifa2026-vm-rg -n vm-front --no-wait
az vm deallocate -g fifa2026-vm-rg -n vm-back  --no-wait
az vm deallocate -g fifa2026-vm-rg -n vm-db    --no-wait
```

Quando quiser voltar: `az vm start -g fifa2026-vm-rg -n vm-front` (idem para as outras). Os IPs privados são preservados; o **IP público pode mudar** se você não tiver pago por um Static IP — anote o novo antes de testar.

#### 7.3 🧹 Apagar tudo (final do evento)

Se acabou e você não quer mais nada:

```bash
az group delete --name fifa2026-vm-rg --yes --no-wait
```

Apaga em bloco: 3 VMs + 3 discos + 3 NICs + 2 NSGs + 1 VNet + 1 Public IP. **Custo zero a partir deste comando.**

---

## 📊 6. Tabela de variáveis e segredos

**Anotações que você fez ao longo do guia** (mantenha no seu bloco de notas, fora do Git):

| Onde | Nome | Exemplo / origem |
|---|---|---|
| 🔐 | *VM Admin Password* | Você escolheu na Fase 2 (login `tftecadmin` em todas as VMs) |
| 🔐 | *SQL SA Password* | Você escolheu no setup do SQL Server (Fase 3.2) |
| 🔐 | *DB_PASSWORD* | Senha do login `fifa2026_db` (Fase 3.6) — vai no `.env` |
| 🔢 | *IP_FRONT* | Public IP da `vm-front` (Fase 2, passo 2) — usado para acessar o app |
| 🔢 | *IP_FRONT_PRIVADO* | Private IP da `vm-front` — usado em ASGs futuras |
| 🔢 | *IP_BACK* | Private IP da `vm-back` (Fase 2, passo 3) — vai no `.env` e no `web.config` |
| 🔢 | *IP_DB* | Private IP da `vm-db` (Fase 2, passo 4) — vai no `.env` |

**No arquivo `C:\inetpub\wwwroot\fifa2026-api\.env` (na `vm-back`):**

`DB_SERVER` · `DB_PORT` · `DB_USER` · `DB_PASSWORD` · `DB_NAME` · `PORT` · `HOST` · `JWT_SECRET` · `JWT_EXPIRES_IN` · `FRONTEND_URL`

**No arquivo `C:\inetpub\wwwroot\fifa2026-web\web.config` (na `vm-front`):**

A regra `<action type="Rewrite" url="http://<IP_BACK>:3001/api/{R:1}" />` é gerada automaticamente no `npm run build` quando você define `BACKEND_URL=http://<IP_BACK>:3001`. **Não edite à mão.**

> 🔒 **Regra de ouro:** segredo nunca vai para o código nem para o repositório. Aqui ficam **no arquivo `.env` da `vm-back`** e **na sua memória** (senhas de admin). _Evolução opcional:_ trocar `.env` por **Azure Key Vault** com Managed Identity nas VMs.

---

## 🛡️ 7. Evolução de segurança (o "VAR" da arquitetura)

> 🧠 **Tópico de aprendizado — não é passo do workshop.** O ambiente que você montou **funciona e ensina os princípios**. Mas todo arquiteto pergunta: *"o que ainda falta para produção de verdade?"*

**O que esse cenário VM já faz bem:**

- 🧅 **Defesa em profundidade real:** apenas a `vm-front` é pública. Back e db **não têm IP público** — Internet não fala com elas, ponto.
- 🧗 **Padrão jump host:** você administra back/db **através** da front, em vez de expor 3389 ao mundo.
- 🔁 **Proxy reverso "raiz":** ARR + URL Rewrite no IIS substituem o que App Service faz nativamente — você entende o mecanismo, não a abstração.

**O que um time de produção ainda endureceria:**

1. **Azure Bastion** em vez de RDP via vm-front — Bastion é serviço gerenciado para acesso a VMs privadas via browser, com auditoria, MFA e zero portas RDP expostas. (Custo: ~$87/mês — só vale se for ambiente de longa duração.)
2. **Migrar para PaaS** — exatamente o caminho do outro guia, [`GUIA-EVENTO.md`](GUIA-EVENTO.md). Você troca 3 VMs Windows por 2 Web Apps + Azure SQL: -80% de custo, -90% de manutenção, +TLS automático, +deploy via `git push`.
3. **Segredos no Key Vault** com Managed Identity — `DB_PASSWORD` e `JWT_SECRET` deixam de estar em texto plano no `.env`.
4. **Patches do OS automatizados** — Azure Update Manager para Windows Server. Em VM, **isso é problema seu**.
5. **Backups** — Azure Backup para as VMs (snapshot diário) ou Always On Availability Groups se quiser HA do SQL.

> 🆚 **Comparação direta — VM × PaaS (mesmo app):**
>
> | Aspecto | Cenário VM (este guia) | Cenário PaaS ([GUIA-EVENTO.md](GUIA-EVENTO.md)) |
> |---|---|---|
> | Custo/mês | ~$90 (24/7) ou ~$15 (com `deallocate`) | ~$18 |
> | Tempo de setup | ~2h30 (1ª vez) | ~1h30 (1ª vez) |
> | Patches do OS | Você gerencia | Microsoft gerencia |
> | TLS | Você instala certificado | Embutido |
> | Deploy | Copiar arquivos via RDP | `git push` → GitHub Actions |
> | Isolamento | NSG + Jump host | Access Restriction |
>
> **Ambos rodam o mesmo código.** A escolha é trade-off entre **controle** (VM) e **velocidade/custo** (PaaS).

---

> 🏁 _Documento vivo — atualizado conforme o evento se aproxima (URL do repo público, dataset do bacpac, contagens). **Bola rolando!**_ ⚽🏆
