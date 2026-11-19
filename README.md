# Site de Agendamento — Pet Shop

Site público onde os **clientes agendam e reagendam horários** (pet shop).

> **Importante:** este projeto é apenas o **site de agendamento** (lado do cliente). O **painel administrativo** (agenda, clientes, PDV, relatórios, configurações) fica em **outra pasta/projeto**: [`adm-petshop`](../adm-petshop). Os dois se conectam ao **mesmo projeto Firebase (Firestore)**.

```
 Site de agendamento (este)  ──►  Firebase / Firestore  ◄──  Painel ADM (adm-petshop)
   cliente agenda/reagenda        mesmo projeto, mesmas        profissionais, serviços, horários,
   lê serviços e horários         coleções                     bloqueios e visualização da agenda
```

- O **ADM cadastra** profissionais, serviços, horários e exceções → o **site lê** e monta a tela.
- O **site grava** o agendamento → o **ADM mostra** na Agenda em tempo real.

## Funcionalidades

- Página institucional com serviços e localização
- Lista de profissionais carregada **100% do Firestore**, a partir do cadastro feito no ADM (nome, foto, WhatsApp, coleção de reservas). **Nenhuma imagem de profissional fica neste projeto**
- Fluxo de agendamento: nome/sobrenome/telefone → data → serviço → horário → confirmação
- Horários livres calculados pela **duração do serviço** (passo de 30 min), descontando reservas existentes (conflito por intervalo)
- Respeita horário de funcionamento, dias de folga, fim de expediente do profissional e **exceções/bloqueios** definidos no ADM
- **Reagendamento**: o cliente busca pelo nome/sobrenome e move o agendamento (transação, com fallback em batch)

## Tecnologias

- HTML, CSS e JavaScript puro (ES Modules), **sem build e sem `npm install`**
- Firebase **Firestore** (SDK 10.12.2 via CDN `gstatic`), sem Authentication (o cliente não faz login)
- Boxicons (ícones) via CDN

## Estrutura

| Arquivo | Função |
|---|---|
| `index.html` | Layout, seções e modais |
| `script.js` | Lógica do site + **`firebaseConfig`** + leitura/gravação no Firestore |
| `style.css` | Estilos |
| `assets/` | Logo e imagens institucionais (fotos dos profissionais **não** ficam aqui; vêm do Firebase) |

## Como sincronizar com o Firebase

Os dois projetos só conversam se usarem **o mesmo `firebaseConfig`** (mesmo `projectId`).

### 1. Projeto Firebase
O site já aponta para o projeto **`petshop-90f21`** (bloco `firebaseConfig` do `script.js`). Para usar outro projeto: [Firebase Console](https://console.firebase.google.com/) → *Project settings → Your apps → Web* → copie o `firebaseConfig` e substitua no `script.js`. Ative o **Firestore Database**.

### 2. Configurar o painel ADM com o mesmo projeto
No projeto [`adm-petshop`](../adm-petshop), abra `firebase.js` e cole **o mesmo objeto `firebaseConfig`** deste `script.js` (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`). O `projectId` precisa ser idêntico nos dois.

### 3. Cadastrar os dados no ADM (o site depende deles)
> **Profissionais não são cadastrados no site.** O site apenas lê a coleção `profissionais` do Firebase: cada cartão (nome, foto, botões de agendar/reagendar) é gerado a partir do cadastro feito no ADM. Para incluir, editar, trocar a foto ou remover um profissional, faça no ADM; o site reflete na próxima abertura, sem alterar código nem adicionar imagens em `assets/`.

No painel ADM, aba **Configurações**, cadastre:

1. **Profissionais**: nome, **foto** (`fotoUrl`), WhatsApp e coleção de reservas. Sem nome e `colecao` o profissional não aparece no site
2. **Serviços** (nome, valor, tempo em minutos)
3. **Horários de funcionamento** e dias da semana
4. (Opcional) **Exceções/bloqueios** gerais ou por profissional

### 4. Coleções e campos usados pelo site

| Caminho no Firestore | Uso no site | Campos lidos |
|---|---|---|
| `profissionais` | lê | `nome`, `fotoUrl`/`foto`, `whatsapp`/`wa`, `colecao` (ou `slug` → `reservas_<slug>`), `fimExpediente`, `diasFolga` |
| `servicos` | lê | `nome`, `valor`/`preco`, `tempoMin`/`tempo`/`duracaoMin` |
| `config/horarios` | lê | `inicio`, `fim` |
| `config/semana` | lê | `weekdays` (0=dom … 6=sáb), `dayStart`, `dayEnd` |
| `config/excecoes/dias/{AAAA-MM-DD}` | lê | `fechado`, `inicio`, `fim` (exceção geral do dia) |
| `profissionais/{id}/excecoes/{AAAA-MM-DD}` | lê | `fechado`, `inicio`, `fim` (exceção do profissional; tem prioridade) |
| `reservas_<profissional>/{AAAA-MM-DD_HH:mm}` | **lê, cria, move** | `data`, `hora`, dados do cliente (`clienteNome`, `clienteSobrenome`, `clienteTelefone`…), `servicoNome`, `servicoValor`, `servicoTempoMin`, `createdAt` |

Prioridade da disponibilidade: **exceção do profissional > exceção geral > semana**.

> O ID do documento (`data_hora`) impede dois agendamentos no mesmo horário. Se o ADM criar agendamentos/bloqueios, deve usar o **mesmo padrão de ID e a mesma coleção** do profissional.

### 5. Regras do Firestore
O site não usa login, então as regras precisam permitir, **sem autenticação**: leitura de `profissionais` (e `excecoes`), `servicos` e `config/**`; leitura, criação e atualização/remoção (reagendamento) em `reservas_*`. Publique as regras no Firebase Console (*Firestore → Rules*).

> ⚠️ O `firebaseConfig` é público no navegador. Restrinja as coleções de gestão (PDV, despesas, clientes…) ao e-mail do administrador (Authentication) e deixe público apenas o que o site precisa.

### 6. Executar

Por usar ES Modules, abra por um servidor local (não por `file://`):

```bash
npx serve .
# ou
python -m http.server 5500
```

Ou use a extensão **Live Server** do VS Code. Para publicar: Vercel, Netlify, Firebase Hosting ou qualquer hospedagem estática.

## Checklist de integração

- [ ] Firestore ativado no projeto Firebase
- [ ] `firebaseConfig` do `script.js` **idêntico** ao `firebase.js` do ADM
- [ ] Regras do Firestore publicadas
- [ ] Profissionais (com `colecao` e foto), serviços e horários cadastrados no ADM
- [ ] Teste: agendar pelo site e conferir na **Agenda** do ADM
- [ ] Teste: bloquear um dia/horário no ADM e conferir que fica indisponível no site

## Observações

- As fotos dos profissionais vêm do campo `fotoUrl` do cadastro no ADM; não é preciso colocar imagens em `assets/`.
- `MAPA_FIM_EXPEDIENTE` e `DIAS_FOLGA` no `script.js` são valores de reserva por nome; o que vale é o cadastrado em `profissionais`.
- As consultas usam um único campo (`data`, `clienteNomeLower`), então **não exigem índice composto**.
