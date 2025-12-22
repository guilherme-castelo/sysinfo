# Central de Análise de TI — Inventário e Dashboard (Sysinfo)

Sistema corporativo de **inventário e monitoramento** para estações e servidores Windows, composto por:

- **Coletor PowerShell** (`GPO-AQS-COMPLETE-SYSINFO.ps1`) que gera **JSON por host** e mantém um **manifesto** central.
- **Dashboard Web** (`index.html`, `script.js`, `style.css`) que consome `manifest.json` + `machines/*.json` e exibe indicadores, alertas e detalhes por máquina.

---

## Objetivos

- Inventariar **hardware, software, rede, segurança e eventos** com coleta automatizada.
- Disponibilizar um **painel único** com visão do parque (dashboard), lista de máquinas e alertas.
- Padronizar a saída em **JSON**, facilitando integrações e exportações.

---

## Componentes

### 1) Coletor (PowerShell)
Arquivo: `GPO-AQS-COMPLETE-SYSINFO.ps1`

Gera, por padrão, no `RepoRoot`:

- `machines/<HOSTNAME>.json` (inventário detalhado do host)
- `manifest.json` (índice consolidado das máquinas)
- `.manifest.lock` (lock para atualização segura do manifesto)

Principais blocos coletados no JSON por host:

- **OS** (caption, build, arquitetura, boot/uptime)
- **Computer / BIOS / BaseBoard**
- **CPU / RAM** (inclui módulos)
- **GPU / Monitor** (EDID + WinForms screens)
- **Storage** (volumes e discos)
- **Network** (IPv4, MACs, detalhes de adaptadores)
- **Temps** (ACPI, discos e sensores quando disponíveis)
- **Processes / Services / Software**
- **EventLogs** (Application/System – críticos/erros)
- **Security** (antivírus, firewall, etc.)
- **IssuesWarn / IssuesCrit** (alertas calculados pelo coletor)

### 2) Dashboard Web
Arquivos: `index.html`, `script.js`, `style.css`

Views principais:
- **Dashboard**: indicadores e gráficos gerais do parque.
- **Máquinas**: cards/grade com busca, filtros e modal com detalhes.
- **Alertas**: consolidação de alertas a partir dos dados coletados.

Recursos adicionais:
- Exportação **JSON/CSV** diretamente pelo navegador (a partir dos dados carregados).
- Modo **produção vs. exemplo** via `config.json`.

---

## Estrutura recomendada do `RepoRoot` (compartilhamento/pasta)

A forma mais simples é manter **dashboard + dados** no mesmo diretório servido via HTTP:

```text
\\SERVIDOR\share\sysinfo\
├── GPO-AQS-COMPLETE-SYSINFO.ps1
├── index.html
├── script.js
├── style.css
├── config.json
├── manifest.json              (gerado/atualizado pelo script)
├── .manifest.lock             (gerado pelo script)
└── machines\
    ├── PC001.json
    ├── PC002.json
    └── ...
```

Arquivos opcionais para laboratório/demonstração:
- `config_exemple.json` (fallback se `config.json` falhar)
- `manifest_exemple.json` (usado quando `AMBIENTE_PRODUCAO=false`)

---

## Pré-requisitos

- Windows PowerShell **5.1**
- Permissões para leitura de informações locais (recomendável **Administrador** para coleta completa)
- Permissão de escrita no `RepoRoot` (especialmente em compartilhamento UNC)

---

## Configuração rápida

### 1) Defina o `RepoRoot`
Por padrão, o script usa o caminho configurado no parâmetro:

- `-RepoRoot "SEU_CAMINHO_AQUI"`

Você pode ajustar para outro UNC ou pasta local.

### 2) Configure o `config.json`
Arquivo: `config.json`

Exemplo:
```json
{
  "AMBIENTE_PRODUCAO": true
}
```

Comportamento:
- `true`  → dashboard carrega `manifest.json`
- `false` → dashboard carrega `manifest_exemple.json`

### 3) Execute o coletor (teste manual)
Exemplo (salvando no compartilhamento padrão):
```powershell
powershell.exe -ExecutionPolicy Bypass -File .\GPO-AQS-COMPLETE-SYSINFO.ps1
```

Exemplo (salvando em pasta local):
```powershell
powershell.exe -ExecutionPolicy Bypass -File .\GPO-AQS-COMPLETE-SYSINFO.ps1 -RepoRoot "C:\sysinfo"
```

### 4) Sirva o dashboard via HTTP
A partir do `RepoRoot`:

**Opção A (Python):**
```powershell
py -m http.server 8080
```

**Opção B (Node/http-server):**
```powershell
npx http-server . -a 0.0.0.0 -p 8080
```

Acesse:
- `http://localhost:8080`

Observação: servir via HTTP evita bloqueios de CORS que ocorrem ao abrir `index.html` diretamente pelo `file://`.

---

## Parâmetros do coletor (PowerShell)

Principais parâmetros disponíveis (conforme `param()` do script):

- `-RepoRoot <caminho>`: destino dos arquivos (`manifest.json`, `machines\*.json`)
- `-ModoColeta "Completo" | "Minimo"`: controla o nível de coleta
- `-IntervaloExecucao <segundos>`: se > 0, entra em loop (inventário periódico)
- **Limiar de alertas**:
  - `-MinMemFreePercent`, `-MinMemFreeGB`
  - `-MinDiskFreePercent`, `-MinDiskFreeGB`
  - `-HighTempWarnC`, `-HighTempCritC`
  - `-MaxProcessCPU`, `-MaxProcessMemoryMB`
- `-SkipTemps`: pula coleta de temperaturas/sensores
- `-DisableJSON`: desabilita escrita do JSON (use apenas se houver outro destino de persistência fora deste fluxo)
- `-EnableRemoteActions`: reservado para cenários com ações remotas (a depender da evolução do projeto)
- Lock do manifesto:
  - `-LockMaxTries` (padrão 60)
  - `-LockSleepMs` (padrão 500)

Exemplo com limiares customizados:
```powershell
powershell.exe -ExecutionPolicy Bypass -File .\GPO-AQS-COMPLETE-SYSINFO.ps1 -MinMemFreePercent 10 -MinDiskFreePercent 10 -HighTempWarnC 75 -HighTempCritC 85
```

---

## Implantação corporativa

### Via GPO (Startup Script)
Recomendação:
1. Armazene o script no compartilhamento (ex.: `\\SERVIDOR\share\sysinfo\GPO-AQS-COMPLETE-SYSINFO.ps1`)
2. Garanta permissões de escrita para o contexto de execução (tipicamente **conta do computador** / `Domain Computers`) no `RepoRoot`
3. Configure a GPO para executar no startup com `ExecutionPolicy Bypass`

Exemplo de linha única (startup):
```powershell
powershell.exe -ExecutionPolicy Bypass -File "\\SERVIDOR\share\sysinfo\GPO-AQS-COMPLETE-SYSINFO.ps1" -RepoRoot "\\SERVIDOR\share\sysinfo"
```

---

## Estrutura de dados (contrato)

### `manifest.json`
Exemplo (campos típicos):
```json
[
  {
    "Hostname": "PC001",
    "Json": "machines/PC001.json",
    "TimestampUtc": "2025-12-22T15:10:20.123Z",
    "Status": "OK",
    "OS": "Windows 10 Enterprise",
    "CollectionMode": "Completo"
  }
]
```

### `machines/PC001.json`
Exemplo (campos principais):
```json
{
  "Hostname": "PC001",
  "TimestampUtc": "2025-12-22T15:10:20.123Z",
  "Status": "OK",
  "IssuesWarn": [],
  "IssuesCrit": [],
  "CollectionMode": "Completo",
  "ScriptVersion": "2.4",
  "OS": {
    "Caption": "Windows 10 Enterprise",
    "Version": "10.0.19045",
    "Build": "19045",
    "Architecture": "64-bit",
    "LastBoot": "2025-12-22T10:00:00.000Z",
    "Uptime": "05:10:20"
  },
  "CPU": { "Name": "Intel(R)...", "Cores": 8, "Logical": 16 },
  "RAM": { "TotalGB": 32, "FreeGB": 12.5, "FreePercent": 39.1 },
  "GPU": { "Resolution": "1920x1080", "RefreshRate": "60Hz", "MaxRefreshRate": "144Hz" },
  "Monitor": { "Count": 2, "Monitors": [], "WinFormsScreens": [] },
  "Storage": { "Volumes": [], "Disks": [] },
  "Network": { "IPv4": [], "MACs": [], "Adapters": [] },
  "Temps": { "ACPI_MaxC": 0, "Disk_MaxC": 0, "MaxC": 0 },
  "Processes": [],
  "Services": [],
  "Software": [],
  "EventLogs": [],
  "Security": {}
}
```

Observação: os exemplos acima ilustram o **formato geral**. A estrutura real pode conter campos adicionais conforme disponibilidade no host.

---

## Performance e carregamento do dashboard

O dashboard utiliza cache-busting (`bust()`) + `fetch` com `no-store` para reduzir efeitos de cache em ambiente web.

Para ambientes com muitas máquinas, a rotina de carga pode ser ajustada para trabalhar com **concorrência limitada** (pool de requisições), evitando o carregamento totalmente sequencial e reduzindo o tempo total.

---

## Troubleshooting

### “Erro ao carregar manifesto”
- Confirme que `manifest.json` está no mesmo diretório servido do dashboard.
- Execute o coletor ao menos uma vez para gerar `manifest.json` e `machines\*.json`.
- Verifique permissões de escrita no `RepoRoot`.

### “Nenhuma máquina encontrada”
- `manifest.json` vazio ou inexistente.
- Falha de escrita no compartilhamento (ACL/SMB).
- Execução do script sem privilégios (coleta incompleta pode resultar em `Status` e dados reduzidos).

### Problemas de cache / dados antigos
- O dashboard já inclui mecanismos anti-cache, mas proxies podem interferir.
- Recomenda-se servir o conteúdo via HTTP e evitar abrir via `file://`.

### Lock do manifesto
- Se houver disputa de escrita (múltiplas máquinas ao mesmo tempo), o script usa `.manifest.lock`.
- Em caso de travamento por lock (antivírus, IO lento), ajuste `-LockMaxTries` e `-LockSleepMs` ou verifique bloqueios no compartilhamento.

---

## Segurança e boas práticas

- Mantenha o compartilhamento `RepoRoot` restrito (somente leitura para usuários do dashboard; escrita apenas para contas necessárias).
- Para execução via GPO, preferir contexto **SYSTEM** e ACL controlada para `Domain Computers`.
- Sempre valide em ambiente de teste antes de expandir para todo o parque.

---

## Versão / Atualizações

- **ScriptVersion:** 2.4 (campo `ScriptVersion` no JSON por host)
- **Última atualização deste README:** 22/12/2025
