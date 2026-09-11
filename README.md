# CRM Lime

CRM comercial da Agência Lime. Página estática única, dados no Firestore.

Controla dois tipos de negócio que têm funil, valor e sazonalidade diferentes:
contrato de agência (fee mensal de mídia, tráfego e social) e cota de patrocínio
do Festival Interlagos. Registra quem indicou cada negócio, porque indicação é o
canal de entrada da casa. E, principalmente, impede que um negócio esfrie sem
ninguém perceber.

## Como funciona por dentro

| Peça | Onde |
|---|---|
| A página | `index.html`, arquivo único, GitHub Pages |
| Os dados | Firestore do projeto `crm-lime` (id `crm-lime-ag`), região São Paulo |
| As regras | `REGRAS-CRM-LIME.txt`, coladas inteiras no console |
| O código-fonte | `build/`, pedaços numerados |
| Montar | `./montar.sh` junta `build/` em `index.html` |
| A especificação | `SPEC.md`, 1.122 linhas, com a formula de cada número |

Nunca edite `index.html` na mão. Edite o pedaço em `build/` e rode `./montar.sh`.

### Conferir antes de publicar

```bash
./montar.sh && python3 conferir.py && python3 testar.py
```

`conferir.py` lê o `index.html` montado e procura violação dos invariantes:
escuta ao vivo no banco, transação, exclusão de verdade, escrita fora da porta
única, campo do usuário indo pra tela sem escape, travessão no texto. Ele sai com
código 1 quando acha erro, então serve para travar a publicação.

`testar.py` é o teste de fumaça. Não existe navegador nem node nesta máquina, mas
o macOS traz um motor de JavaScript em `osascript -l JavaScript`. O arquivo monta
um DOM de mentira, carrega o sistema inteiro e roda as contas com uma base de
amostra. São 51 testes. Eles cobrem o que a checagem de sintaxe não pega: função
com nome errado que só quebra ao executar, campo que nasce indefinido e vira
`NaN` numa soma, sinal invertido em conta de dia, e tela que estoura com a base
vazia, que é exatamente o primeiro dia de uso.

Quatro testes existem por causa de defeitos que já aconteceram e não podem
voltar: leitura de valor com ponto de milhar, limpeza do próximo passo cumprido,
tarefa de avisar quem indicou nascendo pela origem, e a linha do tempo recebendo
registro quando o negócio fecha pela virada de edição.

## Perfis de acesso

A matriz de permissão vive no código, em `build/03-const.js`. O banco guarda só
quem tem qual perfil. Assim ninguém ganha poder editando dado, só o admin
mudando o perfil de alguém.

| Perfil | O que faz |
|---|---|
| Admin | Tudo, mais o painel de Acessos |
| Vendedor | Opera o CRM inteiro. Não cria login |
| Gestor | Lê tudo, define meta, registra Nota. Não move card, não fecha negócio, não edita valor |

O gestor registra Nota de propósito. O dono vai encontrar o contato de patrocínio
num evento e vai querer deixar registrado. Bloquear isso empurra a informação pro
WhatsApp e ela nunca chega ao CRM. Para desligar, troque `atividades:"nota"` por
`atividades:"ver"` no perfil gestor.

## O que este sistema NÃO protege

Leia isso antes de cadastrar o primeiro cliente.

Não existe Firebase Auth com e-mail e senha. O que existe é login anônimo mais
uma senha que decide o perfil. Na prática, **quem tem o endereço da página
consegue falar com o banco e ler tudo**: nome de cliente, valor de contrato,
telefone de decisor e motivo de perda. A matriz de perfis é desenho de tela, não
fechadura.

O que fazer com isso:

1. **Não guarde no CRM o que não pode vazar.** Nada de CPF, CNPJ, dado bancário,
   contrato anexado ou comentário pessoal sobre pessoa. A régua é simples: se
   você não mandaria num grupo de WhatsApp com 50 pessoas, não escreve aqui.
2. A página já tem `noindex`. O repo precisa de `robots.txt` com `Disallow: /`.
3. Crie o alerta de orçamento no Google Cloud do projeto.
4. Migre pra Firebase Auth de verdade no dia em que entrar dado pessoal de
   terceiro em volume, alguém sair da empresa, ou entrar alguém de fora. A
   matriz de perfis e o porteiro do código não mudam uma linha nessa migração.
   Muda o gate, e a regra passa a usar `request.auth.uid`.

As regras do Firestore garantem outra coisa, e essa elas garantem bem: formato
certo, texto com teto de tamanho, carimbo de tempo sempre vindo do servidor, tipo
do negócio imutável, autoria de atividade não reescrita, e **nada apagável por
ninguém**. Exclusão é `arquivado: true`.

## O projeto no Firebase

O CRM tem projeto **próprio**, separado do festival. O motivo não foi plano nem
cota: os dois estão no mesmo Blaze, na mesma conta de faturamento, dividindo o
mesmo crédito. O motivo foi acesso.

Acesso ao console é por projeto. Enquanto o CRM morava dentro de
`festival-interlagos-2026`, qualquer pessoa convidada para o evento conseguia
abrir o Firestore e ler nome de cliente, valor de contrato e telefone de decisor.
Regra de segurança não resolve isso, porque o console passa por cima dela. De
quebra, regra é um arquivo por projeto: agora um erro nas regras do CRM não chega
perto da Central do Evento, da mobilidade nem da pesquisa de imprensa.

Estado em 11 de setembro de 2026:

| Item | Valor |
|---|---|
| Projeto | `crm-lime`, id `crm-lime-ag`, organização limeag.com |
| Plano | Blaze, conta de faturamento compartilhada com o festival |
| Alerta de orçamento | R$ 25, avisa em 50%, 70% e 100% |
| Banco | Firestore `(default)`, southamerica-east1, modo de produção |
| Login | Anônimo ativado, sem limpeza automática de contas |
| Domínios autorizados | localhost, os dois do Firebase, e `contato-lab.github.io` |
| Analytics | Desligado |

O domínio do GitHub Pages precisa estar na lista de domínios autorizados do
Authentication. Sem ele o login anônimo é recusado e o CRM abre mas não lê nada.

**Custo de leitura.** O desenho gasta cerca de 225 leituras por dia com os dois
usuários. Isso não é restrição de funcionamento, é escolha de engenharia: escuta
ao vivo em coleção recarrega a coleção inteira a cada abertura de página, e no 4G
isso se sente. As sete regras que sustentam o número estão comentadas no topo de
`build/04-core.js`.

| Ação | Leituras |
|---|---|
| Abrir o app com cache quente | 4 |
| Carga fria (aparelho novo) | cerca de 261 |
| Abrir um negócio nunca aberto | 10 |
| Trocar de aba, filtrar, buscar, ordenar, abrir relatório | 0 |
| Registrar atividade, mover estágio, cadastrar | 0 |

**O alerta de orçamento é o item de segurança mais importante do projeto.** O
banco é aberto a quem tem o endereço, então quem martelar ele gasta crédito de
verdade. O alerta é o que avisa antes de doer.

## Publicar

O Mac não tem credencial do git. O caminho é upload pelo navegador, com o Chrome
já logado.

**Cada upload substitui o arquivo inteiro, não faz merge.** Subir uma cópia velha
apaga calado tudo que entrou depois dela. Antes de editar, sempre baixe a versão
que está no repo agora:

```bash
curl -s "https://raw.githubusercontent.com/contato-lab/crm-lime/main/index.html?cb=$RANDOM" -o base.html
```

Confira a `VERSAO` dela. Se a cópia local não bate, a do repo manda.

## Roteiro de implantação

Cada passo precisa de dono e dia marcado.

1. Criar o repo `crm-lime` no contato-lab, subir `index.html` e `logo-lime.png`,
   ligar o GitHub Pages.
2. Habilitar o provedor **Anônimo** em Authentication, e acrescentar o domínio
   da página em Authentication, Configurações, Domínios autorizados. Sem os dois
   o CRM abre e não lê nada.
3. Colar `REGRAS-CRM-LIME.txt` no console e publicar. Aqui pode colar o arquivo
   inteiro sem medo: o projeto só tem o CRM dentro.
4. A senha de recuperação **não está escrita no código**, só o resumo
   criptográfico dela (`ACESSO_RECUPERACAO.hash` em `build/03-const.js`), do qual
   não se volta para a senha. A senha em si foi entregue ao dono em 11 de setembro
   de 2026 e não está em nenhum arquivo do repositório. Quem perder gera outra:
   `printf 'lime-crm-2026::a-senha-nova' | shasum -a 256` e troca o valor.
5. Conferir o alerta de orçamento do projeto.
6. Criar os acessos reais no painel de Acessos e anotar cada senha no gerenciador
   de senha. A senha não é recuperável, só trocável.
7. Primeira carga: 40 minutos cadastrando **só os negócios abertos agora**. Não
   existe backlog de digitação. Cliente antigo e negócio perdido entram quando
   forem tocados.
8. Baixar o backup em JSON na primeira sexta e repetir uma vez por mês. É o único
   seguro contra apagar uma coleção sem querer no console.

## O que ficou de fora da versão 1

Kanban arrastável, lead scoring, campos MEDDIC, previsão ponderada calibrada,
feed global de atividades, anexo de arquivo, importação de CSV com detector de
duplicado, push no celular, gamificação e tema claro. O motivo de cada corte está
na seção 10 do `SPEC.md`, com o que entrou no lugar.

Dois merecem explicação curta, porque parecem falta e não são:

**Não tem notificação no celular.** Uma página estática não tem servidor pra
acordar ninguém, e fingir que tem é como o projeto morre na terceira semana. O
lembrete sai da agenda do celular: no primeiro login o sistema oferece criar um
evento recorrente às 8h45 com o link do CRM.

**Não tem arrastar no kanban.** Consome a maior parte do tempo de desenvolvimento
e entrega zero informação nova. Pior: no iPhone o toque longo dispara sozinho no
meio de uma rolagem e move o negócio sem ninguém ver. Mover estágio são dois
toques numa folha, e funciona igual no computador e no celular.


## Histórico de decisões grandes

**11/09/2026, projeto próprio no Firebase.** O CRM saiu de
`festival-interlagos-2026` e ganhou `crm-lime-ag`. Feito antes de qualquer
documento ser gravado, então não houve migração. Ver a seção do Firebase acima.

**11/09/2026, senha de recuperação virou hash.** Antes o código guardava a senha
em texto e calculava o resumo na partida, o que num repositório público equivale a
publicar a senha.

**11/09/2026, três correções vindas de auditoria.** O sistema se declarava
autenticado quando a parte de login do Firebase não carregava, e recusava toda
gravação em silêncio. A sessão com o banco não se refazia sozinha, então quem
deixasse a aba aberta por semanas passava a levar recusa sem entender. E as regras
exigiam na edição dois campos que não exigiam na criação, o que deixaria qualquer
documento vindo de fora do app impossível de editar para sempre.

**Arquivos de regra antigos estão em `_historico/` e não podem ser colados.** Um
deles apagaria a pesquisa de imprensa e a mobilidade inteira do projeto do
festival. O aviso está na porta da pasta.
