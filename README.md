# Interview Prep — Homelab Cybersecurity Portfolio
> Perguntas técnicas esperadas em entrevistas de alternance em cybersécurité na França.
> Atualizar à medida que novos projetos forem concluídos.

---

## Como usar este documento

- **⭐ Básico** — pergunta esperada em qualquer entrevista, mesmo para perfis júnior
- **⭐⭐ Intermediário** — pergunta esperada em empresas com processo técnico estruturado (Wavestone, Thales, Orange Cyberdefense)
- **⭐⭐⭐ Avançado** — pergunta esperada em entrevistas técnicas aprofundadas ou para posições mais sênior

Para cada pergunta: leia, feche o documento, tente responder em voz alta em francês. Depois compare com a resposta sugerida.

---

## Pergunta geral — presente em todas as entrevistas

**⭐ "Fale sobre seus projetos de homelab."**

> Resposta sugerida: Estruture em 3 partes — contexto, o que construiu, o que aprendeu.
> "J'ai construit un homelab de cybersécurité composé de plusieurs projets interconnectés : un moniteur de réseau qui détecte les nouveaux appareils et envoie des alertes Telegram, un stack de monitoring avec Grafana et Prometheus, un serveur DNS sécurisé avec Pi-hole qui bloque plus de 467 000 domaines malveillants, et un pipeline DevSecOps avec GitHub Actions qui scanne automatiquement les vulnérabilités. L'objectif était de couvrir à la fois les aspects défensifs — détection, monitoring, filtrage — et les pratiques DevSecOps modernes."

---

## Projeto 1 — Network Watcher

### Conceitos cobertos
`nmap` · `ARP scanning` · `MAC address` · `systemd` · `Bash scripting` · `REST API` · `asset inventory`

---

**⭐ "O que é um ARP scan e por que o nmap precisa de root para capturar MACs?"**

> O ARP (Address Resolution Protocol) mapeia endereços IP para endereços MAC na rede local. O nmap precisa de privilégios root para criar raw sockets — sockets de baixo nível que permitem enviar e receber pacotes ARP diretamente na camada 2 do modelo OSI. Sem root, o nmap consegue detectar hosts via TCP/IP mas não resolve os MACs porque não tem acesso à camada de enlace.

---

**⭐ "Qual a diferença entre endereço IP e endereço MAC? Por que você monitora o MAC e não o IP?"**

> O IP é um endereço lógico, atribuído dinamicamente pelo DHCP — o mesmo dispositivo pode ter IPs diferentes em momentos distintos. O MAC é um endereço físico, gravado no hardware da interface de rede, teoricamente único e permanente. Monitoro o MAC porque é um identificador mais confiável para rastrear dispositivos: se um celular reconecta e recebe um IP diferente, o MAC continua igual e o sistema reconhece que é o mesmo dispositivo.

> **Ponto de atenção:** MAC randomization — dispositivos modernos (iOS 14+, Android 10+) rotacionam o MAC para proteger a privacidade do usuário. Isso pode gerar falsos positivos no monitor. Vale mencionar isso na entrevista como limitação conhecida.

---

**⭐⭐ "Como você tornaria esse sistema mais robusto em um ambiente corporativo?"**

> Algumas melhorias: (1) Integrar com uma base de dados de ativos conhecidos (CMDB) em vez de um arquivo texto simples. (2) Adicionar correlação com o Active Directory para identificar o dono do dispositivo pelo MAC. (3) Implementar alertas com severidade — dispositivo desconhecido é crítico, dispositivo conhecido reconectando é informativo. (4) Usar SNMP para coletar informações adicionais dos switches sobre qual porta o dispositivo está conectado.

---

**⭐⭐ "O que é um serviço systemd do tipo oneshot vs simple? Qual você usou e por quê?"**

> O tipo `simple` mantém o processo rodando continuamente — o systemd considera o serviço ativo enquanto o processo existir. O tipo `oneshot` executa o processo, aguarda ele terminar, e considera o serviço como concluído. Usei `simple` porque o script tem um loop `while true` que roda indefinidamente, verificando a rede a cada 30 segundos. Um `oneshot` terminaria após a primeira execução.

---

**⭐⭐⭐ "Como esse projeto se relaciona com o conceito de asset inventory em frameworks como o CIS Controls?"**

> O CIS Controls lista o inventário de ativos de hardware como o controle número 1 — você não pode proteger o que não sabe que existe. O Network Watcher implementa a detecção automática de novos dispositivos, que é a base de qualquer programa de asset management. Em um SOC real, isso alimentaria um CMDB e geraria tickets automáticos para validação pelo time de segurança. A correlação MAC → proprietário → autorização é o passo seguinte na maturidade do controle.

---

## Projeto 2 — Network Dashboard

### Conceitos cobertos
`Prometheus` · `Grafana` · `node_exporter` · `SNMP` · `Docker Compose` · `métricas` · `séries temporais` · `observabilidade`

---

**⭐ "O que é o Prometheus e como ele coleta métricas?"**

> O Prometheus é um sistema de monitoramento baseado em pull — ao contrário de sistemas push onde os agentes enviam dados, o Prometheus vai ativamente buscar métricas nos exporters a cada intervalo configurado (15 segundos no nosso caso). Ele armazena tudo como séries temporais: pares (timestamp, valor) identificados por nome da métrica e labels. O modelo pull facilita a detecção de hosts que param de responder — se o exporter não está acessível, o Prometheus sabe imediatamente.

---

**⭐ "O que é o node_exporter e quais métricas ele expõe?"**

> O node_exporter é um agente que expõe métricas do sistema operacional Linux em formato Prometheus via HTTP na porta 9100. Ele lê diretamente de `/proc` e `/sys` — os sistemas de arquivos virtuais do kernel que expõem informações do sistema. Métricas incluem: CPU por core e modo (user, system, iowait), memória (total, disponível, cache, swap), disco (leitura/escrita, latência, uso por partição), rede (bytes, pacotes, erros por interface) e carga do sistema.

---

**⭐⭐ "Por que você usou `pid: host` no node_exporter? Quais são as implicações de segurança?"**

> O `pid: host` faz o container compartilhar o namespace de processos com o host, permitindo que o node_exporter leia `/proc` do sistema real e não do container. Sem isso, as métricas de CPU e memória refletiriam apenas o container, não o host. A implicação de segurança é que o container tem visibilidade sobre todos os processos do host — em produção isso precisaria ser avaliado contra o modelo de ameaça. Uma alternativa mais segura é montar `/proc` como volume read-only, que foi o que fizemos com `- /proc:/host/proc:ro`.

---

**⭐⭐ "O que é SNMP e por que ele não funcionou com o roteador da Vivo?"**

> SNMP (Simple Network Management Protocol) é um protocolo para monitoramento e gerenciamento de dispositivos de rede. Ele expõe uma árvore de informações chamada MIB (Management Information Base) que contém dados como tráfego por interface, tabela ARP, informações de hardware. O roteador da Vivo Fibra é um CPE (Customer Premises Equipment) administrado pela operadora — o firmware é bloqueado para impedir acesso a funcionalidades avançadas como SNMP, provavelmente por razões de suporte e segurança operacional da operadora.

---

**⭐⭐⭐ "Qual a diferença entre monitoramento baseado em métricas e monitoramento baseado em logs? Quando usar cada um?"**

> Métricas são agregações numéricas ao longo do tempo — ideais para detectar tendências, anomalias de volume e disparar alertas baseados em threshold (CPU > 90% por 5 minutos). São eficientes em armazenamento e query. Logs são eventos individuais com contexto completo — ideais para investigação forense, entender o que exatamente aconteceu em um incidente. Em segurança, métricas são usadas para detecção (volume anormal de conexões), e logs para investigação (quais IPs, quais comandos). O stack ideal combina os dois: Prometheus para métricas e ELK ou Wazuh para logs — que é exatamente o que o roadmap do homelab cobre.

---

## Projeto 3 — Secure DNS (Pi-hole)

### Conceitos cobertos
`DNS` · `resolução recursiva` · `blocklists` · `threat intelligence` · `DHCP` · `C2 detection` · `Pi-hole`

---

**⭐ "Explique como funciona a resolução DNS do momento que você digita um domínio até receber a resposta."**

> (1) O navegador verifica o cache local. (2) Se não encontrar, consulta o resolver configurado no sistema — no nosso caso, o Pi-hole em `127.0.0.1`. (3) O Pi-hole verifica se o domínio está na sua blocklist. Se estiver, retorna `0.0.0.0` imediatamente — bloqueando a conexão antes mesmo de ela ser tentada. (4) Se não estiver bloqueado, o Pi-hole encaminha a query para o upstream DNS (1.1.1.1 ou 8.8.8.8). (5) O upstream resolve recursivamente — consultando os root servers, depois os TLD servers, depois o authoritative server do domínio. (6) A resposta volta pelo mesmo caminho e é cacheada no Pi-hole.

---

**⭐ "Por que o Pi-hole é relevante para segurança além de bloquear anúncios?"**

> Do ponto de vista de segurança, o DNS é um vetor crítico: a maioria dos malwares usa DNS para comunicação C2 (command and control). O Pi-hole com blocklists de threat intelligence (como a URLhaus da abuse.ch que adicionamos) bloqueia domínios C2 conhecidos antes que qualquer conexão seja estabelecida — mesmo que o malware já esteja rodando no dispositivo. Além disso, o query log do Pi-hole é uma fonte de visibilidade: qualquer dispositivo da rede que tentar acessar um domínio suspeito aparece nos logs com IP de origem, horário e domínio consultado.

---

**⭐⭐ "O que é a abuse.ch URLhaus e por que você a adicionou como blocklist?"**

> A URLhaus é um projeto da abuse.ch — uma organização suíça de threat intelligence — que coleta e distribui indicadores de comprometimento (IOCs) de campanhas de malware ativas. A lista de hostnames que adicionamos contém domínios atualmente sendo usados para distribuir malware. A diferença em relação às listas genéricas como a StevenBlack é que a URLhaus é atualizada em tempo quase real com ameaças ativas, enquanto as outras cobrem domínios de ads e trackers históricos. Em um SOC real, usar múltiplas fontes de threat intelligence com diferentes especializações é uma prática padrão.

---

**⭐⭐ "Como o Pi-hole funciona como DNS para toda a rede sem instalar nada nos dispositivos?"**

> Configuramos o roteador para distribuir o IP do Pi-hole como servidor DNS via DHCP. Quando um dispositivo conecta na rede e solicita um endereço IP, o roteador responde com o IP, máscara, gateway e — via opção DHCP 6 — o endereço do servidor DNS. O dispositivo usa esse DNS automaticamente sem nenhuma configuração manual. É por isso que a configuração no roteador é mais escalável que configurar DNS manualmente em cada dispositivo.

---

**⭐⭐⭐ "Quais são as limitações do bloqueio DNS como controle de segurança?"**

> Várias limitações importantes: (1) DNS over HTTPS (DoH) — aplicativos que usam DoH fazem as queries diretamente para servidores como `1.1.1.1` via HTTPS na porta 443, bypassando o Pi-hole completamente. (2) Hardcoded DNS — malwares sofisticados ignoram o DNS do sistema e usam IPs diretos ou seus próprios resolvers. (3) Domínios gerados algoritmicamente (DGA) — malwares que geram domínios aleatoriamente dificilmente estarão em blocklists. (4) Falsos positivos — listas agressivas podem bloquear domínios legítimos. Por isso o bloqueio DNS é uma camada de defesa, não a única — Defense in Depth.

---

## Projeto 4 — DevSecOps Pipeline

### Conceitos cobertos
`CI/CD` · `GitHub Actions` · `Trivy` · `Gitleaks` · `pip-audit` · `CVE` · `CVSS` · `secrets management` · `shift left security` · `vulnerability management`

---

**⭐ "O que é 'shift left security' e como esse pipeline implementa esse conceito?"**

> Shift left significa mover as verificações de segurança para mais cedo no ciclo de desenvolvimento — da direita (produção) para a esquerda (desenvolvimento). Tradicionalmente, segurança era verificada depois que o código estava pronto para deploy. Com o pipeline que construímos, toda vez que um desenvolvedor faz push, as verificações rodam automaticamente antes de qualquer merge. Isso reduz o custo de correção — uma vulnerabilidade detectada no commit é muito mais barata de corrigir do que uma detectada em produção.

---

**⭐ "O que é um CVE e o que significa o score CVSS?"**

> CVE (Common Vulnerabilities and Exposures) é um identificador único para vulnerabilidades conhecidas, mantido pelo MITRE. Cada CVE tem um score CVSS (Common Vulnerability Scoring System) de 0 a 10 que quantifica a severidade com base em fatores como: vetor de ataque (rede vs. local), complexidade de exploração, privilégios necessários, impacto em confidencialidade, integridade e disponibilidade. No pipeline configuramos para bloquear apenas HIGH (7.0-8.9) e CRITICAL (9.0-10.0) — LOW e MEDIUM geram aviso mas não bloqueiam o deploy.

---

**⭐ "Por que o Gitleaks faz checkout com `fetch-depth: 0`?"**

> Por padrão, o GitHub Actions faz um shallow clone com apenas o último commit para ser mais rápido. Se um desenvolvedor commitou um token no commit 50, depois "removeu" no commit 51, um shallow clone só veria o commit 51 e não detectaria o leak. O `fetch-depth: 0` busca o histórico completo do repositório, permitindo ao Gitleaks varrer todos os commits. Em Git, deletar um arquivo não apaga do histórico — o conteúdo permanece acessível para sempre.

---

**⭐⭐ "Qual a diferença entre Trivy e pip-audit? Por que usar os dois?"**

> O Trivy escaneia a imagem Docker completa — camada a camada — incluindo o OS base (Alpine), binários do sistema e pacotes Python. É uma visão holística do container. O pip-audit foca exclusivamente nas dependências Python declaradas no `requirements.txt`, usando a base de dados PyPA (Python Packaging Advisory Database) que tem cobertura específica do ecossistema Python. Na prática encontramos vulnerabilidades diferentes: o Trivy detectou CVEs no Alpine e em dependências internas do pip, enquanto o pip-audit identificou CVEs diretamente no Flask e Requests. Usar os dois garante cobertura mais completa.

---

**⭐⭐ "Como você tomou a decisão de aceitar alguns CVEs em vez de corrigi-los?"**

> O processo foi: (1) Identificar o componente vulnerável — `zlib` no Alpine, `jaraco.context` e `wheel` que são dependências internas do pip. (2) Avaliar a exploitabilidade no contexto da aplicação — o CVE do zlib afeta o utilitário `untgz`, que não é usado pela aplicação Flask em runtime. Os CVEs do jaraco e wheel afetam o processo de build, não o container em execução. (3) Documentar a decisão no `.trivyignore` com justificativa explícita. Aceitar um risco sem documentar é negligência. Aceitar com justificativa e registro é gestão de risco — é o que empresas maduras fazem.

---

**⭐⭐ "O que você faria se o Gitleaks detectasse um token no histórico do repositório?"**

> Procedimento de resposta: (1) Revogar o token imediatamente na plataforma onde foi gerado — GitHub, AWS, qualquer serviço. Assumir que foi comprometido, independente de o repositório ser público ou privado. (2) Investigar os logs de uso do token para verificar se houve acesso não autorizado. (3) Remover do histórico do Git com `git filter-branch` ou `git-filter-repo` — mas lembrar que se o repositório é público ou foi clonado, o token já pode ter sido capturado. (4) Implementar controles preventivos: variáveis de ambiente, secrets managers (AWS Secrets Manager, HashiCorp Vault), pre-commit hooks com Gitleaks local.

---

**⭐⭐⭐ "Como você integraria esse pipeline num ambiente corporativo com múltiplos times e repositórios?"**

> Algumas práticas enterprise: (1) Centralizar as definições de workflow em um repositório de templates reutilizáveis — cada time herda o pipeline de segurança sem precisar mantê-lo. (2) Usar uma política de branch protection que exige que todos os jobs de segurança passem antes de permitir merge. (3) Integrar os resultados com uma plataforma de gestão de vulnerabilidades (Defect Dojo, por exemplo) para tracking histórico. (4) Configurar notificações para o time de segurança quando CVEs críticos são detectados. (5) Implementar exceções com prazo — um CVE aceito hoje precisa ser reavaliado em 30/60/90 dias.

---

## Perguntas transversais — sobre o portfólio como um todo

**⭐⭐ "Como esses projetos se conectam numa estratégia de Defense in Depth?"**

> Cada projeto cobre uma camada diferente: o Pi-hole filtra na camada DNS antes de qualquer conexão ser estabelecida. O nftables controla o tráfego na camada de rede — o que entra e sai do host. O Network Watcher monitora a camada de acesso — quem está na rede. O Grafana + Prometheus dá visibilidade sobre o comportamento do sistema. O pipeline DevSecOps protege a cadeia de desenvolvimento. Nenhuma camada é suficiente sozinha — um malware com DoH bypassa o Pi-hole, mas o nftables pode bloquear a porta de C2, e o Wazuh (próximo projeto) detectaria o comportamento anômalo nos logs.

---

**⭐⭐ "Qual projeto você considera mais relevante para uma posição de SOC analyst e por quê?"**

> Para SOC analyst, o Wazuh SIEM é o mais diretamente relevante — centraliza logs, gera alertas e é a ferramenta que um analista usa no dia a dia. Mas o que diferencia o portfólio é a integração: o Network Watcher gera eventos, o Pi-hole gera logs DNS, o nftables gera logs de bloqueio — tudo isso alimenta o Wazuh para correlação. Um SOC analyst que entende não só a ferramenta mas como os dados chegam até ela é muito mais valioso.

---

**⭐⭐⭐ "Se você fosse apresentar esse homelab para justificar um investimento em ferramentas similares para uma empresa, como você estruturaria o argumento?"**

> Estruturaria em três eixos: visibilidade, controle e resposta. Visibilidade: hoje a empresa sabe quais dispositivos estão na rede? Tem logs de DNS? Tem métricas de infraestrutura? Se não, o Network Watcher, Pi-hole e Grafana demonstram o valor de ter essa visibilidade com custo zero de licença. Controle: o nftables e Pi-hole mostram que é possível implementar controles de rede efetivos sem appliances caros. Resposta: o pipeline DevSecOps demonstra que segurança pode ser integrada ao desenvolvimento sem bloquear a produtividade. O argumento financeiro é: o custo de detectar uma brecha depois é ordens de magnitude maior do que o custo de prevenção.

---

## Próximos projetos — perguntas que serão adicionadas

- [ ] Firewall avançado — nftables (em progresso)
- [ ] IDS — Suricata
- [ ] SIEM — Wazuh
- [ ] Honeypot — Cowrie
- [ ] AWS IAM lab
- [ ] AWS GuardDuty

---

*Atualizado em: Abril 2026*
*Projetos concluídos: 4/10*
