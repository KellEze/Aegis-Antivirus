# AEGIS — Next-Gen Security

Sistema completo de antivírus educacional, em duas partes que conversam entre si:

| Parte | O que é | Tecnologia |
|---|---|---|
| `server/` | Site de vendas, cadastro/login, painel do cliente, checkout (simulado), licenças e API | Python, Flask, SQLite |
| `desktop/` | **Aplicativo antivírus**: verificação, quarentena, tempo real, atualização de assinaturas | Python, Tkinter (somente biblioteca padrão) |
| `tests/` | 32 testes automatizados (site, API, motor, quarentena, tempo real e integração app ↔ servidor) | unittest |

## Como rodar o site

```bash
cd server
python -m venv .venv
# Windows: .venv\Scripts\activate      Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
python app.py
```

Abra **http://127.0.0.1:5000**. O banco (`instance/aegis.db`) e a chave secreta (`instance/secret.key`) são criados automaticamente na primeira execução.

Variáveis de ambiente opcionais: `AEGIS_PORT`, `AEGIS_HOST`, `AEGIS_DEBUG=1`, `AEGIS_SECRET_KEY`, `AEGIS_PUBLIC_URL` (endereço público, usado no zip do app), `AEGIS_HTTPS=1` (cookie seguro quando houver HTTPS).

## Como rodar o aplicativo

**Pelo site (recomendado):** crie uma conta, abra **Baixar o app**, baixe o zip (já vem apontando para o seu servidor) e execute `run_windows.bat` (Windows) ou `./run_linux.sh` (Linux/macOS).

**Direto da pasta do projeto:**

```bash
cd desktop
python aegis_app.py        # interface gráfica
python aegis_cli.py --help # linha de comando
```

No Linux, o Tkinter pode precisar de: `sudo apt install python3-tk`.

**Gerar o .exe (Windows):** `desktop\build_exe.bat` → `desktop\dist\AEGIS.exe`.

## Teste rápido de ponta a ponta (5 minutos)

1. Suba o site e **crie uma conta** (plano Essencial, grátis).
2. Em **Baixar o app**, baixe e abra o aplicativo; entre com e-mail e senha (ou a chave de licença).
3. No app, **Painel → Criar arquivos de teste**. Responda "Sim" para verificar a pasta.
4. O arquivo `eicar_test.txt` é detectado e **movido para a quarentena**. O `aegis_update_test.txt` **não** é detectado ainda.
5. Clique em **Atualizar assinaturas** e verifique de novo: agora o segundo arquivo também é detectado — a atualização realmente muda o que o app enxerga.
6. No site, **Minha área → Histórico**: a verificação e as ameaças aparecem.
7. Na aba **Proteção** do app, o tempo real está bloqueado. No site, assine o **Plus** (cartão de teste `4242 4242 4242 4242`), volte ao app (ele atualiza o plano sozinho ao abrir) e ative a proteção: coloque um arquivo EICAR na pasta Downloads e veja a detecção automática.

> O EICAR é um arquivo de teste inofensivo. Se o seu outro antivírus (ex.: Windows Defender) apagá-lo antes do AEGIS, é normal.

## O que o site faz

- Cadastro e login com senha em hash, **sessão**, proteção **CSRF**, limite de tentativas contra força bruta, cabeçalhos de segurança e CSP.
- **Checkout em nova aba** para os planos pagos, com validação (algoritmo de Luhn, validade, CVV). Pagamento **simulado**: guarda só bandeira e 4 últimos dígitos.
- Painel: visão geral, **chave de licença**, **dispositivos** (limite por plano), **histórico** de verificações e ameaças, compras, troca de senha, cancelamento de assinatura, exportação CSV (Pro).
- Assinaturas com validade de 30 dias; ao vencer, a conta volta sozinha ao Essencial.
- API para o aplicativo (`/api/auth/login`, `/api/me`, `/api/signatures`, `/api/scans`).

| Plano | Preço | Dispositivos | Tempo real | Histórico | CSV |
|---|---|---|---|---|---|
| Essencial | Grátis | 1 | — | 5 | — |
| Plus | R$ 19/mês | 3 | ✓ | 50 | — |
| Pro | R$ 39/mês | 10 | ✓ | 50 | ✓ |

## O que o aplicativo faz de verdade

- **Verificação rápida** (Downloads, Desktop, temporários) e **personalizada** (qualquer pasta).
- **Assinaturas SHA-256** e **padrões de bytes**, atualizáveis pelo servidor.
- **Heurística** com pontuação: dupla extensão enganosa (`fatura.pdf.exe`), executável disfarçado de imagem/documento, scripts com comandos codificados/de download, tentativa de apagar cópias de sombra ou desligar o Defender, entropia alta em executáveis, `autorun.inf`. 35+ pontos = suspeito; 70+ = malicioso.
- **Quarentena reversível**: o arquivo é codificado e isolado; dá para restaurar ou excluir.
- **Proteção em tempo real** (Plus/Pro): monitora pastas e verifica arquivos novos depois que terminam de chegar (varredura a cada 3 s).
- Envia o resultado ao painel online (apenas nome do arquivo, nunca o conteúdo ou o caminho).
- Funciona **offline** com as assinaturas já baixadas.

## Limites (leia antes de apresentar)

- **É um projeto educacional.** A base de assinaturas inclui o arquivo de teste EICAR e um de demonstração; **não** há base de malware real. A heurística é uma boa demonstração, mas gera falsos positivos/negativos e **não substitui um antivírus comercial**. O site foi escrito de acordo (sem estatísticas ou promessas inventadas).
- **Pagamento simulado.** Para cobrar de verdade, integre um gateway (Stripe, Mercado Pago…) no `comprar()` do `app.py`.
- **Tempo real por varredura periódica**, não por driver de kernel: não bloqueia a execução do arquivo antes de ele abrir.
- Para colocar online: use HTTPS (`AEGIS_HTTPS=1`), defina `AEGIS_SECRET_KEY`, rode atrás de um proxy e use um banco mais robusto se houver muitos usuários.
- A interface gráfica foi desenvolvida e revisada sem acesso a um desktop com Tkinter. Toda a lógica (motor, quarentena, API, tempo real, controlador) tem testes; a janela em si deve ser conferida visualmente na sua máquina.

## Testes

```bash
pip install -r server/requirements.txt
python -m unittest discover tests -v
```

## Estrutura

```
aegis-antivirus/
├── server/
│   ├── app.py            rotas do site, painel, checkout e API
│   ├── database.py       SQLite (users, devices, compras, scans, detections)
│   ├── security.py       CSRF, rate limit, validações, Luhn
│   ├── signatures.json   base de assinaturas servida ao app
│   ├── templates/  static/
│   └── requirements.txt
├── desktop/
│   ├── aegis_app.py      interface gráfica (Tkinter)
│   ├── aegis_cli.py      linha de comando
│   ├── controller.py     lógica do app (compartilhada)
│   ├── engine.py         motor de detecção
│   ├── quarantine.py  realtime.py  api_client.py  config.py
│   └── run_*.{bat,sh}  build_exe.{bat,sh}  assets/
└── tests/
```
