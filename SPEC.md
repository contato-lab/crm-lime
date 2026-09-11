> # CRM COMERCIAL DA AGÊNCIA LIME, ESPECIFICAÇÃO DE IMPLEMENTAÇÃO
>
> Versão 1. Arquivo único `index.html` estático, GitHub Pages, repo novo no contato-lab. HTML, CSS e JS puro. Firebase compat via CDN (`firebase-app-compat.js`, `firebase-auth-compat.js`, `firebase-firestore-compat.js`). Firestore do projeto `festival-interlagos-2026`, região `southamerica-east1`. Prefixo `crm_` em toda coleção.
>
> Este documento é a fonte única. Onde as seis propostas se contradiziam, a decisão está registrada e a versão descartada não aparece aqui.

---

## 1. O QUE É E PRA QUEM

Um CRM de duas pessoas que controla dois tipos de negócio (contrato de agência e cota de patrocínio do Festival Interlagos), registra quem indicou cada negócio e, principalmente, impede que um negócio esfrie sem ninguém perceber.

**Quem usa**
- **Vendedor** (opera): cadastra, move estágio, registra atividade, fecha. Abre na tela Hoje.
- **Gestor** (acompanha): lê tudo, registra nota, define meta. Não move card, não fecha negócio, não edita valor. Abre no Painel.
- **Admin**: o vendedor com o painel de Acessos ligado. Cria e revoga login.

**A única disciplina obrigatória do sistema:** não dá pra fechar um toque sem marcar o próximo passo com data, ou encerrar o negócio. Todas as outras obrigações foram cortadas pra essa aguentar em pé.

**O que o sistema promete e cumpre**
1. Uma fila diária que cabe na tela e nunca acusa.
2. O texto pronto pra mandar no WhatsApp.
3. Quatro números que o gestor confere sozinho, cada um com a lista dos negócios que entram na conta.
4. O ranking de quem traz cliente.

**O que ele não faz:** ler mensagem recebida, avisar sozinho com o app fechado, sincronizar e-mail, guardar arquivo, prever com IA, pontuar lead.

---

## 2. MODELO DE DADOS

### 2.1 Arquitetura de leitura, a decisão que manda em tudo

**Coleção de verdade, sincronização incremental por carimbo do servidor, cache em `localStorage`.** Nada de documento-índice único com todos os negócios dentro, nada de `onSnapshot` em coleção.

- Cada negócio, contato, tarefa e atividade é um documento próprio.
- Todo documento tem `at`, gravado com `FieldValue.serverTimestamp()`.
- A carga seguinte pede só o que mudou: `where("at", ">", Timestamp.fromMillis(ultimoSync - 300000))`. Uma consulta que não devolve nada custa 1 leitura.
- O cache local guarda negócios, contatos, tarefas e config. **Atividade nunca entra no cache geral**, porque é onde mora texto grande e é o que estoura os 5 MB do `localStorage` no Safari.
- Nada é apagado de verdade. Exclusão é `arquivado:true`, que é uma escrita e portanto viaja no delta. Delete físico sumiria do delta e deixaria o cache mentindo pra sempre.

**Por que não o documento-índice único** (proposto por duas dimensões): uma requisição sobrescreve o mapa inteiro e apaga a base comercial; toda edição reescreve o array inteiro (100 KB de upload por clique no 4G do estacionamento); duas abas do mesmo usuário se sobrescrevem em silêncio. O delta por coleção custa praticamente o mesmo em regime (3 a 4 leituras por abertura contra 1) e não tem nenhum desses três problemas. O preço é a carga fria de um aparelho novo (cerca de 260 leituras, algumas vezes por mês), e esse preço está no orçamento da seção 9.

**Sem `enablePersistence`.** O cache é o nosso, no `localStorage`. Escrita que falha ou sai offline vai pra `crm_outbox` no `localStorage` e é reenviada na abertura seguinte e no evento `online`. Isso existe porque com persistência ligada a tela diz "salvo" antes de o servidor confirmar, e o iOS descarrega a aba no bolso levando a escrita junto.

### 2.2 Convenções que valem em todo documento

| Coisa | Formato | Motivo |
|---|---|---|
| Dinheiro | `number` inteiro, em reais, sem centavos | Ninguém negocia cota em R$ 179.999,47. Mata float e soma que vaza centavo |
| Data de calendário | `string "AAAA-MM-DD"` | Compara lexicograficamente, não tem fuso, cabe no cache sem conversão |
| "Há quanto tempo" | também `string "AAAA-MM-DD"` | Dia basta pra "16 dias sem contato". Elimina relógio adiantado do celular |
| `at` (carimbo de sincronismo) | `Timestamp` do servidor | É o único campo Timestamp do sistema. Convertido pra ms na leitura (`.toMillis()`) antes de ir pro cache |
| `ts` (desempate na timeline) | `number` ms do cliente | Só ordena dentro do mesmo dia |
| Id | gerado no cliente, com prefixo | `n_` negócio, `c_` contato, `t_` tarefa, `a_` atividade |
| Texto normalizado | minúsculo, sem acento | Campo `busca` e campos `*Chave`, gravados na escrita |

```js
function idNovo(p){ return p + "_" + Date.now().toString(36) + Math.random().toString(36).slice(2,5); }
function chave(s){ return String(s||"").normalize("NFD").replace(/[̀-ͯ]/g,"").toLowerCase().trim(); }
function hojeISO(){ var d=new Date(); d.setHours(12,0,0,0);
  return d.getFullYear()+"-"+("0"+(d.getMonth()+1)).slice(-2)+"-"+("0"+d.getDate()).slice(-2); }
function diasVencido(dataISO){ return diasEntre(dataISO, hojeISO()); }   // positivo = vencido
function diasEntre(a,b){ return Math.round((new Date(b+"T12:00:00") - new Date(a+"T12:00:00"))/864e5); }
```

`diasVencido` existe com esse nome porque as duas propostas originais inverteram o sinal de `dias(hoje, data)` e todo negócio em dia ficaria vermelho no primeiro dia de uso.

### 2.3 Lista final de coleções

| Caminho | O que guarda | Cresce | Quando é lido |
|---|---|---|---|
| `crm_negocios/{id}` | Um negócio, dos dois tipos | ~130 no ano 1 | Carga fria e delta |
| `crm_contatos/{id}` | Pessoa. Contato e indicador são a mesma coisa | ~120 | Carga fria e delta |
| `crm_tarefas/{id}` | Toque que não é negócio (avisar indicador, retomar, renovação) | ~10 abertas | Carga fria e delta, `where feita == false` |
| `crm_atividades/{id}` | Uma linha por interação | ~2.000/ano | Só ao abrir um negócio, `limit(10)` |
| `crm_acessos/{id}` | Login, com hash da senha | 3 a 6 | Só no login |
| `crm_config/geral` | Metas | 1 | Carga e delta |

Não existe `crm_empresas`, `crm_indicadores`, `crm_index`, `crm_espelho`, `crm_metricas`, `crm_meta`, `crm_auditoria`, `crm_log`. Os motivos estão na seção 10.

### 2.4 `crm_negocios/{id}`

| Campo | Tipo | Obrig | Exemplo |
|---|---|---|---|
| `id` | string | sim | `"n_m9x2k4a7t"` |
| `tipo` | string | sim, imutável | `"agencia"` ou `"patrocinio"` |
| `titulo` | string (<=120) | sim | `"Gestão de mídia 2027"` |
| `empresa` | string (<=120) | sim | `"Marroca Editora"` |
| `empresaChave` | string | sim | `"marroca editora"` |
| `contatoId` | string ou "" | não | `"c_dani"` |
| `contatoNome` | string | não | `"Daniela Prado"` |
| `contatoTel` | string | não | `"5511987654321"` (só dígitos, com DDI) |
| `estagio` | string | sim | `"ag_proposta"` |
| `status` | string | sim | `"aberto"`, `"ganho"`, `"perdido"`, `"encerrado"` |
| `origem` | string | sim | `"indicacao"`, `"inbound"`, `"prospeccao"`, `"cliente_atual"`, `"evento"` |
| `indicadorId` | string ou "" | sim se origem=indicacao | `"c_cezinha"` |
| `indicadorNome` | string | idem | `"Cezinha de Madureira"` |
| `valorRef` | number | sim (pode ser 0) | `42000` |
| `ag` | map ou null | só tipo agencia | ver abaixo |
| `pt` | map ou null | só tipo patrocinio | ver abaixo |
| `prevFechamento` | string data | sim | `"2026-10-30"` |
| `prevEstimada` | bool | sim | `true` enquanto for a data automática |
| `proxTipo` | string | sim se aberto | `"cobrar_prop"` |
| `proxTexto` | string (<=140) | não | `"falar com o Marcos, não com a Paula"` |
| `proxData` | string data | sim se aberto | `"2026-09-16"` |
| `proxHora` | string ou "" | não | `"15:00"` (só em `prox Tipo = reuniao`) |
| `congeladoAte` | string data ou "" | não | `"2026-11-10"` |
| `congeladoMotivo` | string | não | `"fora do ciclo de budget"` |
| `vezesAdiado` | number | sim | `2` |
| `vezesRemarcouPrev` | number | sim | `3` |
| `ultimoToqueEm` | string data | sim | `"2026-09-09"` |
| `ultimoToqueTxt` | string (<=140) | sim | `"Reunião com a Daniela, fee aprovado"` |
| `estagioDesde` | string data | sim | `"2026-09-02"` |
| `maiorEstagio` | number | sim | `2` (maior índice já alcançado) |
| `motivoPerda` | string ou "" | obrig se perdido | `"sem_verba"` |
| `motivoPerdaObs` | string (<=400) | não | `"Adiaram pro orçamento de 2028"` |
| `fechadoEm` | string data ou "" | obrig se ganho/perdido | `"2026-09-04"` |
| `encerradoEm` | string data ou "" | obrig se encerrado | `"2027-10-31"` |
| `motivoEncerramento` | string ou "" | obrig se encerrado | `"renovado"` |
| `linkProposta` | string (<=300) | não | `"https://drive.google.com/..."` |
| `obs` | string (<=1000) | não | `"Não ligar antes das 10h"` |
| `busca` | string | sim | `"marroca editora gestao de midia daniela prado"` |
| `criadoEm` | string data | sim | `"2026-08-11"` |
| `criadoPor` | string (<=60) | sim | `"Eric"` |
| `alteradoPor` | string (<=60) | sim | `"Eric"` |
| `at` | Timestamp servidor | sim | |
| `arquivado` | bool | sim | `false` |

**`ag`, só quando `tipo == "agencia"`:**

| Campo | Tipo | Exemplo |
|---|---|---|
| `mrr` | number (R$/mês) | `3500` |
| `mrrInicial` | number, gravado uma vez e nunca sobrescrito | `4500` |
| `meses` | number, 0 = indeterminado | `12` |
| `verbaMidia` | number (R$/mês) | `15000` |
| `pctVerba` | number (%), padrão 0 | `0` |
| `servicos` | list<string> (<=6) | `["trafego","social"]` |
| `inicioContrato` | string data | `"2026-11-01"` |
| `fimContrato` | string data ou "" | `"2027-10-31"` |
| `renovacaoDeId` | string ou "" | `"n_k7h2m1x"` |
| `quemDecide` | string (picklist) | `"o dono"` |
| `faixaVerba` | string (picklist) | `"2 a 5 mil"` |

**`pt`, só quando `tipo == "patrocinio"`:**

| Campo | Tipo | Exemplo |
|---|---|---|
| `edicao` | string (chave da tabela `EDICOES`) | `"moto_2027"` |
| `cota` | string | `"master"` |
| `valor` | number (dinheiro) | `180000` |
| `valorInicial` | number, gravado uma vez | `220000` |
| `permuta` | number, nunca somada ao dinheiro | `60000` |
| `entregaveis` | string (<=600), uma linha por item | `"Ativação 6x6\nMenção do locutor 3x/dia"` |
| `quemAprova` | string | `"Diretoria de marketing"` |
| `mesBudget` | string ou "" | `"2026-11"` |

**Regras de valor, uma função só, chamada em todo ponto de escrita:**

```js
function mrrDe(n){                                  // R$/mes que a Lime fatura
  if(n.tipo !== "agencia") return 0;
  return Math.round((n.ag.mrr||0) + (n.ag.verbaMidia||0) * ((n.ag.pctVerba||0)/100));
}
function valorRefDe(n){                             // R$ do negocio, pra ordenar e listar
  if(n.tipo === "patrocinio") return Math.round(n.pt.valor||0);   // permuta fica fora
  return Math.round(mrrDe(n) * ((n.ag.meses||0) || 12));          // indeterminado = 12 meses, dito na tela
}
```

`valorRef` é o único campo derivado que fica gravado, porque ordenar e somar o funil não pode depender de ler documento nenhum. Todo o resto (probabilidade, valor ponderado, dias parado, temperatura, ranking) é calculado na hora, em memória.

**Verba de mídia não é receita.** Ela aparece em linha própria, rotulada "verba do cliente, não é receita da Lime". Só entra no MRR a fatia de `pctVerba`, e a Carteira ativa mostra as duas partes separadas (seção 7.1).

### 2.5 `crm_contatos/{id}`

Contato e indicador são a mesma entidade. Criar uma segunda entidade "indicador" faria o Paulo da Shineray existir duas vezes e nunca mais bater.

| Campo | Tipo | Obrig | Exemplo |
|---|---|---|---|
| `id` | string | sim | `"c_cezinha"` |
| `nome` | string (<=120) | sim | `"Cezinha de Madureira"` |
| `nomeChave` | string | sim | `"cezinha de madureira"` |
| `empresa` | string (<=120) | não | `"Shineray do Brasil"` |
| `empresaChave` | string | não | `"shineray do brasil"` |
| `cargo` | string (<=80) | não | `"Gerente de marketing"` |
| `tel` | string (<=20) | não | `"5511999998888"` |
| `email` | string (<=120) | não | `"paulo@shineray.com.br"` |
| `ehIndicador` | bool | sim | `true` |
| `acordo` | string (<=140) | não | `"10% do primeiro mês de fee"` |
| `obs` | string (<=400) | não | `"Responde melhor depois das 18h"` |
| `busca`, `criadoEm`, `criadoPor`, `at`, `arquivado` | | sim | |

**Contato exige nome e pelo menos uma forma de contato** (WhatsApp, telefone fixo ou e-mail), não WhatsApp especificamente. Contato de montadora com ramal e e-mail corporativo é justamente o do maior ticket da casa.

**Não existem contadores de indicação gravados.** O ranking (`nIndicacoes`, `nGanhos`, `valorGerado`) é calculado em memória sobre os negócios que já estão carregados. Custa 0 leitura, nunca desalinha, não precisa de lote atômico nem de botão de recalcular.

### 2.6 `crm_atividades/{id}`

Id com carimbo invertido, pra ordem de nome de documento já ser do mais novo pro mais velho e não precisar de índice composto:

```js
function idAtiv(){ return "a_" + (9999999999999 - Date.now()) + Math.random().toString(36).slice(2,4); }
```

| Campo | Tipo | Obrig | Exemplo |
|---|---|---|---|
| `negocioId` | string ou "" | não | `"n_m9x2k4a7t"` |
| `contatoId` | string ou "" | não | `"c_dani"` |
| `tipo` | string | sim | `ligacao`, `whats`, `reuniao`, `email`, `proposta`, `nota`, `estagio` |
| `texto` | string (<=1200) | sim | `"Fee de 3.500 aprovado, querem começar em novembro"` |
| `quando` | string data | sim | `"2026-09-09"` (o dia em que aconteceu) |
| `ts` | number ms | sim | `1757462400000` (desempate dentro do dia) |
| `por` | string (<=60) | sim | `"Eric"` |
| `de`, `para` | string ou "" | só tipo estagio | `"ag_diagnostico"`, `"ag_proposta"` |
| `at` | Timestamp servidor | sim | |

Timeline do negócio ordena por `quando` desc e, dentro do mesmo dia, por `ts` desc. Quando `quando` é diferente do dia de `ts`, a linha mostra "registrado em 08/09, aconteceu em 05/09".

O cartão do negócio não lê atividade: ele usa `ultimoToqueEm` e `ultimoToqueTxt`, que são gravados no mesmo save.

### 2.7 `crm_tarefas/{id}`

Existe porque três coisas importantes não são negócio e, sem casa, nunca aparecem: avisar quem indicou quando o negócio fecha, retomar um perdido na data combinada, e lembrar da renovação do contrato.

| Campo | Tipo | Exemplo |
|---|---|---|
| `tipo` | string | `avisar_indicador`, `retomar`, `renovacao`, `pedir_indicacao` |
| `texto` | string (<=160) | `"Avisar a Fernanda que fechamos com a Marroca"` |
| `data` | string data | `"2026-09-15"` |
| `contatoId`, `contatoNome`, `contatoTel` | string | pra montar o botão de WhatsApp |
| `negocioId`, `negocioTitulo` | string ou "" | link |
| `feita` | bool | `false` |
| `criadoEm`, `criadoPor`, `at`, `arquivado` | | |

Consulta do boot: `where("feita","==",false).limit(30)`, igualdade simples, sem índice composto. O filtro de data é em memória.

### 2.8 `crm_config/geral`

```
{ metaMrrMes: 5000,                        // R$/mes de MRR novo, alvo mensal
  metaCota: { "moto_2027": 600000 },       // R$ por edicao
  at: <serverTimestamp> }
```

Editável pelo gestor e pelo admin. Sem meta cadastrada, o Painel mostra "sem meta definida" com um link, nunca `NaN%`.

### 2.9 `crm_acessos/{id}`

```
{ nome:"Comercial", perfil:"vendedor", hash:"a3f1...", ativo:true,
  criadoEm:"2026-09-10", criadoPor:"Eric", at:<serverTimestamp> }
```

A senha nunca é gravada. O que se grava é `SHA-256(SAL + senha.trim().toLowerCase())` via `crypto.subtle`, que existe em qualquer navegador em https. O login lê a coleção inteira (3 a 6 leituras, só quando não há sessão válida) e compara hashes em memória.

```js
var SAL = "lime-crm-2026::";
function sha256(txt){
  if(!(window.crypto && crypto.subtle)) return Promise.reject(new Error("sem-crypto"));
  return crypto.subtle.digest("SHA-256", new TextEncoder().encode(SAL + String(txt||"").trim().toLowerCase()))
    .then(function(b){ return Array.prototype.map.call(new Uint8Array(b),
      function(x){ return ("0"+x.toString(16)).slice(-2); }).join(""); });
}
```

Todo caminho que chama `sha256` tem `.catch`, com a mensagem "Não deu pra verificar a senha neste navegador". Sem isso, abrir por `file://` ou por IP da rede (que é como se testa antes de subir) deixa o botão Entrar mudo.

**Senha de recuperação no código, uma só:**

```js
// Funciona SEMPRE, mesmo com a colecao de acessos vazia, apagada ou fora do ar.
// Hash de "limeadmin2026". TROCAR no dia da implantacao (passo 4 da secao 12).
var ACESSO_RECUPERACAO = { nome:"Recuperação", perfil:"admin",
  hash:"c1f0b8...conferir com: printf 'lime-crm-2026::minhasenha' | shasum -a 256" };
```

Quem entra por ela vê faixa vermelha fixa: "Você entrou pela senha de recuperação, que está no código público. Crie seu acesso em Acessos." Só existe uma, e é de admin, porque a segunda e a terceira viram o login do dia a dia e nunca são trocadas.

---

## 3. REGRAS DO FIRESTORE

**Aviso operacional que vale mais que o código:** o projeto tem **um** arquivo de regras. O bloco abaixo entra no mesmo arquivo da Central do Evento, antes do `match /{document=**}` final, e o arquivo inteiro é republicado de uma vez. Publicar só o pedaço do CRM apaga as regras da Central e derruba o evento. Sempre partir da versão que está no console.

**Pré-requisito:** habilitar o provedor **Anônimo** em Authentication no console, e o app chamar `firebase.auth().signInAnonymously()` antes de qualquer leitura. Custa 0 leitura e derruba varredor automático e `curl` sem token. Não afeta a Central, que tem regras próprias em caminhos próprios.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ===== CENTRAL DO EVENTO: os blocos que ja existem entram aqui, intactos =====

    // ===== CRM COMERCIAL LIME ==================================================
    // A matriz de permissao (quem edita o que) vive no codigo, nao aqui: sem
    // Firebase Auth de verdade a regra nao sabe QUEM e voce, so sabe que voce
    // passou pelo Anonymous Auth. O que a regra garante e formato, teto de
    // tamanho, carimbo do servidor e a impossibilidade de apagar.
    function ok(){ return request.auth != null; }
    function carimbado(){ return request.resource.data.at == request.time; }

    match /crm_negocios/{id} {
      allow read: if ok();
      allow create, update: if ok() && carimbado()
        && request.resource.data.keys().size() <= 60
        && request.resource.data.tipo in ['agencia','patrocinio']
        && request.resource.data.status in ['aberto','ganho','perdido','encerrado']
        && request.resource.data.titulo is string && request.resource.data.titulo.size() <= 120
        && request.resource.data.empresa is string && request.resource.data.empresa.size() <= 120
        && request.resource.data.valorRef is int
        && request.resource.data.arquivado is bool
        && (!('obs' in request.resource.data) || request.resource.data.obs.size() <= 1000)
        && (!('motivoPerdaObs' in request.resource.data) || request.resource.data.motivoPerdaObs.size() <= 400)
        && (!('linkProposta' in request.resource.data) || request.resource.data.linkProposta.size() <= 300)
        && (resource == null || request.resource.data.tipo == resource.data.tipo)   // tipo e imutavel
        && (resource == null || request.resource.data.criadoEm == resource.data.criadoEm);
      allow delete: if false;     // exclusao e arquivado:true, senao some do delta
    }

    match /crm_contatos/{id} {
      allow read: if ok();
      allow create, update: if ok() && carimbado()
        && request.resource.data.keys().size() <= 25
        && request.resource.data.nome is string && request.resource.data.nome.size() <= 120
        && (!('obs' in request.resource.data) || request.resource.data.obs.size() <= 400)
        && (!('tel' in request.resource.data) || request.resource.data.tel.size() <= 20)
        && request.resource.data.arquivado is bool;
      allow delete: if false;
    }

    match /crm_tarefas/{id} {
      allow read: if ok();
      allow create, update: if ok() && carimbado()
        && request.resource.data.keys().size() <= 20
        && request.resource.data.texto is string && request.resource.data.texto.size() <= 160
        && request.resource.data.feita is bool;
      allow delete: if false;
    }

    // Colecao que cresce pra sempre: teto de texto e obrigatorio, senao um colar
    // de 200 KB entra e cada abertura de negocio fica cara.
    match /crm_atividades/{id} {
      allow read: if ok();
      allow create: if ok() && carimbado()
        && request.resource.data.keys().size() <= 14
        && request.resource.data.texto is string && request.resource.data.texto.size() <= 1200
        && request.resource.data.tipo is string && request.resource.data.tipo.size() <= 20
        && request.resource.data.por is string && request.resource.data.por.size() <= 60
        && request.resource.data.quando is string && request.resource.data.quando.size() == 10;
      // corrigir o texto pode; reescrever autoria e data nao
      allow update: if ok() && carimbado()
        && request.resource.data.por == resource.data.por
        && request.resource.data.ts == resource.data.ts
        && request.resource.data.texto.size() <= 1200;
      allow delete: if false;
    }

    match /crm_config/{doc} {
      allow read: if ok();
      allow create, update: if ok() && doc in ['geral']
        && request.resource.data.keys().size() <= 10;
      allow delete: if false;
    }

    match /crm_acessos/{id} {
      allow read: if ok();
      allow create, update: if ok()
        && request.resource.data.keys().size() <= 10
        && request.resource.data.perfil in ['admin','vendedor','gestor']
        && request.resource.data.nome is string && request.resource.data.nome.size() <= 80
        && request.resource.data.hash is string && request.resource.data.hash.size() == 64
        && request.resource.data.ativo is bool;
      allow delete: if false;     // revogar e ativo:false, e a sessao cai na proxima carga
    }
  }
}
```

**O que essas regras garantem:** formato certo, texto com teto, `at` sempre do servidor (nenhum celular adiantado cega o delta), `tipo` do negócio imutável, autoria de atividade não reescrita, e nada apagável por ninguém, nem por engano nem de propósito.

**O que elas não garantem, dito com todas as letras:** quem passou pelo Anonymous Auth pode ler tudo e escrever em tudo, inclusive criar um acesso de perfil `admin`. A matriz `PERFIS` é desenho de tela, não fechadura. Consequências práticas, que precisam estar no README do repo:
1. **Não guardar no CRM o que não pode vazar.** Nada de CPF, CNPJ, dado bancário, contrato, ou nota pessoal sobre gente ("eles estão quebrados", "o cara é um chato"). Régua: se você não mandaria num grupo de WhatsApp com 50 pessoas, não escreve aqui.
2. `<meta name="robots" content="noindex,nofollow">` na página e `robots.txt` com `Disallow: /`.
3. **Alerta de orçamento no Google Cloud** no projeto, com data: o crédito de R$ 1.745 vence em 18/11/2026 e o ataque mais barato que existe (encher o banco) derruba a Central junto.
4. Migrar pra Firebase Auth com e-mail e senha no dia em que entrar dado pessoal de terceiro em volume, alguém sair da empresa, ou entrar alguém de fora. A matriz `PERFIS` e o porteiro `motivoBloqueio` não mudam uma linha nessa migração; muda o gate e a regra passa a usar `request.auth.uid`.
5. App Check fica fora da v1 **de propósito**: a imposição é por serviço e vale pro projeto inteiro, então ligar no Firestore obriga a Central a mandar token também. Só com as duas páginas atualizadas juntas e fora de janela de evento.

**Índices:** nenhum composto. As consultas usadas são `where("at",">",ts)` (campo único, automático), `where("feita","==",false)` (automático), `where("negocioId","==",id).limit(10)` (automático, e a ordem de nome de documento já é a ordem cronológica invertida pelo id da atividade).

---

## 4. OS DOIS FUNIS

Estágio e status são coisas diferentes. `ganho`, `perdido` e `encerrado` não são colunas do kanban: se fossem, ao fechar o negócio se perderia a informação de **em que estágio ele morreu**, que é a única pergunta que explica onde o funil vaza, e o quadro viraria depósito de card morto.

Id de estágio é prefixado por funil (`ag_`, `pt_`). Assim um estágio do funil errado aparece como "estágio desconhecido" na tela em vez de cair calado numa coluna que não é dele.

### 4.1 Agência (fee mensal de mídia, tráfego, social)

| # | id | Rótulo | Critério de saída (ato do comprador, verificável) | Prob. | Teto de dias no estágio |
|---|---|---|---|---|---|
| 0 | `ag_contato` | Contato | Reunião marcada com data e hora, aceita pelo cliente | 10% | 7 |
| 1 | `ag_diagnostico` | Diagnóstico | `quemDecide` e `faixaVerba` preenchidos (picklist) e o cliente disse com data quando quer a proposta | 25% | 10 |
| 2 | `ag_proposta` | Proposta em análise | O cliente respondeu qualquer coisa que não seja silêncio: aprovou, pediu desconto, pediu ajuste, marcou devolutiva | 45% | 10 |
| 3 | `ag_negociacao` | Negociação | Acordo verbal nos três números (fee, meses, data de início) e contrato enviado | 75% | 14 |

Quatro estágios, não seis. "Business case" e "procurement" saem: na empresa média brasileira o dono é comprador, pagador e usuário na mesma pessoa, e não existe jurídico fazendo redline de contrato de agência.

O nome do estágio 2 fala do estado do cliente, não do que o vendedor fez. Proposta enviada há 20 dias sem resposta continua em "Proposta em análise", com relógio vermelho. Silêncio não avança nada.

**Entrar em `ag_proposta` exige `ag.mrr > 0`, `ag.meses` (ou "indeterminado") e `prevFechamento` confirmada.** Antes disso valor é opcional, e o cartão mostra "valor a definir" em cinza. Forçar valor no dia 1 gera "R$ 1.000" chutado que envenena a previsão.

### 4.2 Patrocínio (cota do Festival Interlagos)

| # | id | Rótulo | Critério de saída | Prob. | Teto |
|---|---|---|---|---|---|
| 0 | `pt_mapeado` | Mapeado | Uma pessoa nomeada da marca, com cargo, respondeu e aceitou conversar | 5% | 21 |
| 1 | `pt_apresentado` | Apresentação feita | A marca disse o objetivo dela, quem aprova a verba e em que mês o budget é decidido | 15% | 30 |
| 2 | `pt_proposta` | Proposta de cota | A marca respondeu: pediu ajuste de valor ou de entregável, marcou devolutiva, ou disse que vai levar pra aprovação | 30% | 30 |
| 3 | `pt_aprovacao` | Em aprovação da marca | Aprovação confirmada por escrito, com valor. Reprovação vai direto pra Perdido | 60% | 60 |
| 4 | `pt_contrato` | Contrato | Contrato ou PO assinado | 85% | 30 |

O estágio 3 existe porque é onde o negócio some por dois meses e é onde o CRM se paga. Em "Cota proposta" você cobra o contato; em "Em aprovação" você pergunta **quando** é o comitê e agenda em cima da data.

No estágio 3 o relógio de estágio é frouxo (60 dias) e o de toque é apertado (15): é normal a marca demorar, não é normal a gente sumir enquanto ela demora.

### 4.3 Constantes prontas pro código

```js
var FUNIL = {
  agencia: [
    { k:"ag_contato",     rot:"Contato",            p:0.10, teto:7  },
    { k:"ag_diagnostico", rot:"Diagnóstico",        p:0.25, teto:10 },
    { k:"ag_proposta",    rot:"Proposta em análise",p:0.45, teto:10, compromisso:true },
    { k:"ag_negociacao",  rot:"Negociação",         p:0.75, teto:14 }
  ],
  patrocinio: [
    { k:"pt_mapeado",     rot:"Mapeado",            p:0.05, teto:21 },
    { k:"pt_apresentado", rot:"Apresentação feita", p:0.15, teto:30 },
    { k:"pt_proposta",    rot:"Proposta de cota",   p:0.30, teto:30 },
    { k:"pt_aprovacao",   rot:"Em aprovação",       p:0.60, teto:60, compromisso:true },
    { k:"pt_contrato",    rot:"Contrato",           p:0.85, teto:30 }
  ]
};
// dias de silencio tolerados: 2 numeros por funil, nao 9. Ninguem consegue
// explicar por que Proposta tolera 4 e Negociacao tolera 3.
var TOQUE = { agencia:{ normal:5, ultimo:3 }, patrocinio:{ normal:15, ultimo:7 } };
var ABANDONO = { agencia:30, patrocinio:60 };   // sugere fechar como "sumiu"

// Edicoes do festival. Chave PROPRIA do CRM, independente da Central do Evento,
// que usa id "moto" e "auto" sem ano. Quem precisar cruzar usa .edicao + .ano.
var EDICOES = [
  { k:"moto_2027", rot:"Moto 2027", edicao:"moto", ano:2027,
    dataEvento:"2027-08-11", corteComercial:"2027-06-25" },
  { k:"auto_2027", rot:"Auto 2027", edicao:"auto", ano:2027,
    dataEvento:"2027-08-25", corteComercial:"2027-07-09" }
];

var ORIGENS = [["indicacao","Indicação"],["cliente_atual","Cliente atual"],
  ["inbound","Site, DM ou formulário"],["evento","Evento ou feira"],["prospeccao","Prospecção nossa"]];

var MOTIVOS_PERDA = [
  { k:"preco",           rot:"Achou caro",                 reabrir:0,   funil:"ambos" },
  { k:"ficou_com_atual", rot:"Ficou com a agência atual",  reabrir:180, funil:"agencia" },
  { k:"concorrente",     rot:"Foi pra outra agência",      reabrir:0,   funil:"ambos" },
  { k:"sem_verba",       rot:"Sem verba agora",            reabrir:90,  funil:"ambos" },
  { k:"sumiu",           rot:"Sumiu, não respondeu",       reabrir:60,  funil:"ambos" },
  { k:"timing",          rot:"Só na próxima",              reabrir:"informar", funil:"ambos" },
  { k:"reprovado",       rot:"Reprovado na marca",         reabrir:0,   funil:"patrocinio" },
  { k:"janela_evento",   rot:"Perdeu a janela do evento",  reabrir:0,   funil:"patrocinio" },
  { k:"fora_do_perfil",  rot:"Não era perfil (a gente desqualificou)", reabrir:0, funil:"ambos", foraDaTaxa:true }
];

var MOTIVOS_ENCERRAMENTO = ["renovado","fim natural sem renovar","cliente cancelou",
  "a Lime encerrou","cliente fechou as portas"];

var PROX_TIPOS = [
  { k:"ligar",        rot:"Ligar",                       dias:2 },
  { k:"mensagem",     rot:"Mandar mensagem",             dias:1 },
  { k:"reuniao",      rot:"Reunião",                     dias:null, hora:true },
  { k:"enviar_prop",  rot:"Enviar proposta",             dias:3 },
  { k:"cobrar_prop",  rot:"Cobrar resposta da proposta", dias:3 },
  { k:"enviar_contr", rot:"Enviar contrato",             dias:2 },
  { k:"cobrar_aprov", rot:"Cobrar aprovação interna",    dias:15, funil:"patrocinio" },
  { k:"retomar",      rot:"Retomar",                     dias:null }
];
var MOTIVOS_ADIAR   = ["Cliente pediu","Não consegui contato","Dependo de terceiro","Foi minha falha"];
var MOTIVOS_CONGELAR= ["Cliente viajou","Espera aprovação","Fora do ciclo de budget","Sazonal"];
var CONGELAMENTO_MAX = 90;    // dias
var ADIAMENTOS_PARA_RISCO = 3;
var VALOR_PRESUMIDO = { agencia:42000, patrocinio:60000 };  // so pra ordenar negocio sem valor
```

### 4.4 Regras de transição

1. **Pular estágio é permitido** e grava `maiorEstagio` com o índice alcançado. Bloquear faz o vendedor mentir o estágio.
2. **Voltar estágio é permitido**, vira linha na timeline e nada mais. Sem contador no card.
3. **Toda mudança de estágio exige próximo passo novo com tipo e data.** É o pedágio de mover um card, e é o que impede o funil de virar decoração que muda de cor.
4. Toda mudança grava `estagioDesde`, `maiorEstagio` e uma atividade `tipo:"estagio"` com `de` e `para`.
5. **Fechar Ganho exige:** agência, `ag.mrr > 0` e `inicioContrato`; patrocínio, `pt.valor > 0` e `pt.edicao`. Ganho aceita "pagamento confirmado" tanto quanto contrato assinado, porque agência brasileira começa a rodar antes do contrato voltar em metade dos casos.
6. **Fechar Perdido exige `motivoPerda`** (picklist, sem opção "outros", porque "outros" come 60% das respostas em três meses e mata a análise). Motivo com `reabrir` cria uma `crm_tarefa` do tipo `retomar` na data.
7. **Fechar com `origem == indicacao`** cria uma `crm_tarefa` `avisar_indicador` pra amanhã, ganho **ou** perdido. É a mensagem que mais gera a próxima indicação e a que mais se esquece de mandar.
8. **Ganho de agência** cria uma `crm_tarefa` `renovacao` pra 60 dias antes de `ag.fimContrato`, quando houver.
9. **Renovação é negócio NOVO**, `origem: "cliente_atual"`, com `ag.renovacaoDeId` apontando pro anterior. O anterior fecha como `encerrado` com `motivoEncerramento: "renovado"`, e esse motivo **fica fora do MRR perdido**. Sem essa regra, esticar a data faz a maior vitória do trimestre não existir, e encerrar e recriar mostra uma perda de R$ 3.500 que não aconteceu.
10. **Congelar** (`congeladoAte`, máximo 90 dias) tira o negócio das cobranças, deixa o card esmaecido no funil, e **empurra `prevFechamento` pra depois do descongelamento**, avisando na tela ("previsão movida pra dezembro"). Sem válvula de escape honesta o vendedor bota data falsa e o dado inteiro apodrece.
11. **Só quem opera muda estágio.** A checagem roda na hora de desenhar e de novo na hora de gravar.

### 4.5 Semáforo e risco

```js
function temperatura(n){
  if(n.status !== "aberto") return "fechado";
  if(n.congeladoAte && n.congeladoAte > hojeISO()) return "congelado";
  var f = FUNIL[n.tipo], i = idxEstagio(n);
  var venc  = diasVencido(n.proxData);                 // + = vencido
  var silen = diasVencido(n.ultimoToqueEm);
  var parado= diasVencido(n.estagioDesde);
  var limToque = (i === f.length-1) ? TOQUE[n.tipo].ultimo : TOQUE[n.tipo].normal;
  if(venc  >= 1)            return "vermelho";
  if(parado >  f[i].teto*1.5) return "vermelho";
  if(venc  >= -1)           return "amarelo";          // vence hoje ou amanha
  if(parado >  f[i].teto)   return "amarelo";
  if(silen >  limToque)     return "amarelo";
  return "verde";
}
function riscoAlto(n){                                  // so a previsao usa
  return diasVencido(n.proxData) >= 7
      || (n.vezesAdiado||0) >= ADIAMENTOS_PARA_RISCO
      || diasVencido(n.estagioDesde) > FUNIL[n.tipo][idxEstagio(n)].teto*2;
}
```

`vezesRemarcouPrev` é contador separado e **não** alimenta risco: remarcar a previsão três vezes num ciclo de patrocínio de 171 dias é normal, e punir isso ensina o vendedor a nunca mais mexer na data.

---

## 5. MATRIZ DE PERFIS

```js
// ===== PERMISSAO =====
// A matriz vive AQUI, no codigo. O banco guarda so QUEM tem qual perfil.
// Pra mudar o que um perfil pode, sobe arquivo novo. Ninguem ganha poder
// editando dado. Niveis: "nenhum" < "ver" < "nota" < "gerenciar".
var PERFIS = {
  admin: {
    rotulo:"Admin", cor:"lime",
    negocios:"gerenciar", contatos:"gerenciar", atividades:"gerenciar",
    metas:true, acessos:true, exportar:true, home:"hoje"
  },
  vendedor: {
    rotulo:"Vendedor", cor:"lime",
    negocios:"gerenciar", contatos:"gerenciar", atividades:"gerenciar",
    metas:true, acessos:false, exportar:true, home:"hoje"
  },
  gestor: {
    rotulo:"Gestor", cor:"cinza",
    negocios:"ver", contatos:"ver", atividades:"nota",   // registra Nota, e so isso
    metas:true, acessos:false, exportar:true, home:"painel"
  }
};

var ORDEM = { nenhum:0, ver:1, nota:2, gerenciar:3 };
function nivel(c){ return (PERM && PERM[c]) || "nenhum"; }
function aoMenos(c,m){ return ORDEM[nivel(c)] >= ORDEM[m]; }

function podeEditarNegocios(){ return aoMenos("negocios","gerenciar"); }
function podeEditarContatos(){ return aoMenos("contatos","gerenciar"); }
function podeRegistrarNota(){  return aoMenos("atividades","nota"); }
function podeRegistrarTudo(){  return aoMenos("atividades","gerenciar"); }
function podeMexerMeta(){      return !!PERM && PERM.metas === true; }
function ehAdmin(){            return !!PERM && PERM.acessos === true; }

// ---- PORTEIRO: devolve null quando pode, ou o texto do motivo quando nao pode.
// O MESMO texto vai pro toast do bloqueio e pra linha cinza embaixo do botao,
// entao a tela e o bloqueio nunca discordam. Colecao nao listada nasce FECHADA.
function motivoBloqueio(colecao, doc, acao){
  if(!SESSAO || !PERM) return "Sessão encerrada. Entre de novo.";
  if(colecao === "crm_negocios")  return podeEditarNegocios() ? null : "Seu acesso é só de leitura.";
  if(colecao === "crm_contatos")  return podeEditarContatos() ? null : "Seu acesso é só de leitura.";
  if(colecao === "crm_tarefas")   return podeEditarNegocios() ? null : "Seu acesso é só de leitura.";
  if(colecao === "crm_atividades"){
    if(podeRegistrarTudo()) return null;
    if(podeRegistrarNota() && doc && doc.tipo === "nota") return null;
    return "Seu acesso registra só nota.";
  }
  if(colecao === "crm_config")  return podeMexerMeta() ? null : "Seu acesso é só de leitura.";
  if(colecao === "crm_acessos") return ehAdmin() ? null : "Só o Admin mexe em acessos.";
  return "Coleção fora do CRM.";
}

// ---- A UNICA porta de escrita do sistema ----
function gravar(colecao, id, dados, acao){
  var motivo = motivoBloqueio(colecao, dados, acao);
  if(motivo){ toast(motivo); return Promise.reject(new Error(motivo)); }
  dados.alteradoPor = SESSAO.nome;
  dados.at = firebase.firestore.FieldValue.serverTimestamp();
  aplicarNoCache(colecao, id, dados);                       // tela muda na hora
  return db.collection(colecao).doc(id).set(dados, {merge:true})
    .then(function(){ marcarEnviado(colecao, id); })
    .catch(function(e){ paraOutbox(colecao, id, dados, e); marcarPendente(colecao, id); throw e; });
}
```

**Regras de tela**
- Navegação sem permissão **some** (aba Acessos pra quem não é admin).
- Ação sem permissão **fica visível e desabilitada**, com o motivo escrito embaixo em `--text3`. `title` não existe no dedo, e tela sem botão nenhum parece quebrada: o gestor abre um negócio no celular, não vê ação nenhuma, e manda mensagem perguntando se o CRM caiu.
- **O gestor registra Nota e só Nota.** O dono vai encontrar o cara do patrocínio num evento e vai querer deixar registrado. Bloquear isso empurra a informação pro WhatsApp e ela nunca chega ao CRM. Desliga trocando `"nota"` por `"ver"`.

**Sessão:** 7 dias no `localStorage` (`crm_sessao`). Revalidada contra a lista de acessos a cada carga: perfil mudou, atualiza e redesenha; `ativo:false` ou acesso sumido, cai pro gate com o motivo na tela. Se a lista vier **vazia** (erro de leitura, regra republicada errada), **não derruba ninguém**: mantém a sessão e mostra "não consegui ler os acessos" no cabeçalho.

**Painel de Acessos:** lista com nome, perfil, ativo. Ações: trocar senha, trocar perfil, ativar/desativar. Não existe apagar. Bloqueios: não dá pra desativar o próprio acesso nem rebaixar o último admin ativo. Ao criar, a senha gerada (12 caracteres) aparece **num cartão que fica na tela** em Barlow Condensed grande, com botão Copiar e link `wa.me` pronto, e só some quando a pessoa confirma. Senha não é recuperável, só trocável. Checagem de duplicata varre a lista inteira, **inclusive desativados**, senão reativar cria dois logins com o mesmo hash e a autoria passa a depender da ordem do laço.

**Qualquer perfil troca a própria senha** por um item "Minha senha" no menu do cabeçalho. Celular roubado numa sexta à noite não pode depender do admin acordar.

**Limite honesto da autoria:** `criadoPor` e `por` são o nome do acesso, não da pessoa. Senha é compartilhável. Serve pra distinguir duas pessoas que confiam uma na outra, não como prova de nada.

---

## 6. MAPA DE TELAS

Navegação por hash, porque o voltar do Android tem que sair do detalhe sem fechar o app, e porque os dois vão mandar link de negócio um pro outro no WhatsApp, o que num time de duas pessoas substitui metade das notificações que um CRM grande precisa ter.

```
LOGIN (uma senha, que decide o perfil)
└─ APP  cabecalho fixo: logo · BUSCA · [+] · quem sou · pontinho de sync · atualizar · sair
   ├─ HOJE      #/hoje        home do vendedor
   ├─ FUNIL     #/funil/:tipo kanban leitura, seletor [Agência][Patrocínio]
   ├─ NEGÓCIOS  #/negocios    lista com filtros, abertos e fechados
   │  └─ DETALHE #/negocio/:id   a tela mais importante do sistema
   ├─ PESSOAS   #/pessoas     contatos e indicadores, mesma lista
   │  └─ #/pessoa/:id
   ├─ PAINEL    #/painel      home do gestor
   └─ ACESSOS   #/acessos     só admin
```

Abas no topo no desktop (padrão da casa, e a sidebar comeria 240px que o kanban precisa). No celular, barra fixa embaixo: **Hoje · Funil · (+) · Buscar · Mais**.

### 6.1 HOJE (home do vendedor)

É a tela mais importante e não estava em nenhuma proposta como consenso. Com 30 negócios abertos, o kanban não informa nada às 9h da manhã: o vendedor já sabe de cor onde cada um está. O que ele não sabe é **qual passou do ponto**. Em funil de mil leads quem esconde a informação é o volume; em funil de 30 quem esconde é o tempo, e tempo o kanban não mostra.

```
FILA DE HOJE
    5                          Fecha até 30/09: R$ 68.500
(Barlow 900, 64px)

⏰ MARROCA EDITORA                                   R$ 3.500/mês
   Reunião hoje, 14h                                  [zap] [feito]
   último toque 08/09: mandei o escopo de 12 meses
────────────────────────────────────────────────────────────────
🔥 SHINERAY DO BRASIL                                 R$ 180.000
   Cobrar aprovação interna, venceu há 3 dias          [zap] [feito]
   último toque 02/09: pediram pra reapresentar pro comitê
────────────────────────────────────────────────────────────────
💤 MUNDO BRINCA                                      R$ 2.800/mês
   16 dias sem contato, o limite em Diagnóstico é 5    [zap] [feito]
                                              [ver os outros 2]
▌TAREFAS
☐ Avisar a Fernanda que fechamos com a Marroca        [zap] [feita]
▌QUEM TE TROUXE NEGÓCIO             [ver ranking]
```

**O que entra na fila, em ordem**

| # | Balde | Teto | Ordenação |
|---|---|---|---|
| 1 | Próximo passo vencido | 3 | `valorRef` desc, com 1 vaga reservada pro mais recente |
| 2 | Hoje e amanhã | 4 | hora marcada primeiro, depois valor |
| 3 | Tarefas com data vencida ou de hoje | 3 | data |
| 4 | Esfriando (passou do teto de estágio ou de toque) | 2 | valor |
| 5 | Sem próximo passo, criado há mais de 2 dias | 2 | mais recente |

Teto duro de **8 cartões**, e "ver os outros N" com o total honesto ("3 de 19 sem data"). Atraso maior que 14 dias sai da fila e vai pro **Resgate de sexta**, uma aba com teto de 5 por semana. Esse corte é o que impede a terça com 11 atrasados que faz a pessoa fechar a aba e não voltar.

**Negócio sem valor entra na ordenação com `VALOR_PRESUMIDO`, não com zero.** Com zero, a indicação fresca cadastrada em 20 segundos fica sempre no fim e nunca aparece, que é exatamente o lead mais quente da casa.

**Toda linha diz por que está ali, com o número.** Nunca "atrasado", sempre "16 dias sem contato, o limite em Diagnóstico é 5". Alerta sem motivo explícito é treinado a ser ignorado em uma semana.

**Fila vazia é vitória e a tela trata como vitória**, em `--lime`: "Fila limpa. Duas coisas que valem seu tempo", com um negócio parado e um indicador frio (mais de 120 dias sem indicar, com dois ganhos ou mais). As duas sugestões saem do cache, sem leitura.

`hoje()` é recalculado em todo render e no `visibilitychange`. Aba fixada por dias congelava a fila na data de ontem e mostrava "0 pra hoje" numa terça cheia.

### 6.2 FUNIL

Um funil por vez, trocado por pílula segmentada no topo, escolha guardada no `localStorage`. Onze colunas lado a lado é ilegível e impossível no celular; um funil genérico é a gosma que o briefing proíbe, porque "Proposta" não significa a mesma coisa pra um fee de R$ 3.500/mês e pra uma cota de R$ 180.000. A faixa de totais mostra os dois funis sempre, então nada some.

Faixa: `Agência: R$ 249.000 em 9 negócios · R$ 11.800/mês de MRR` e `Patrocínio: R$ 238.000 em 5`. Nunca somados num número só. Fechados do mês aparecem como resumo discreto à direita, nunca como coluna.

**Cartão:** selo de temperatura com o número de dias, EMPRESA em Barlow 700 caixa alta (é o que o vendedor lembra; ninguém procura "Social + tráfego 12 meses"), título em 12px, valor herói em Barlow 900 `--lime` (`R$ 3.500/mês` na agência, `R$ 180.000` no patrocínio), linha de apoio (`R$ 42.000 em 12 meses` ou `MOTO 2027`), chip `via Carla` quando houver indicação, e o próximo passo em uma linha, ou `⚠ sem próximo passo` em `--warn`.

**Mover estágio, caminho único: a folha.** Toque no cartão abre folha de baixo pra cima com a lista de estágios, o atual marcado e os dias no atual; escolher um estágio abre, **na mesma folha**, o bloco de próximo passo já preenchido com a sugestão do tipo e da data; um botão Salvar. Dois toques, com o pedágio pago, e funciona igual no desktop e no celular. **Não existe arrastar na v1**, e não existe atalho de segurar: no iOS o long press dispara sozinho no meio de uma rolagem e move o negócio sem ninguém ver o toast. Toast com **Desfazer de 8 segundos** em toda mudança.

No celular o quadro vira um estágio por vez, com barra de pílulas rolável mostrando nome, contagem e valor de cada estágio.

**Virada de edição:** quando a `dataEvento` passa, a primeira abertura mostra uma faixa com os negócios abertos daquela edição. Quem passou de `pt_mapeado` é resolvido um a um (perder ou migrar pra próxima edição, criando negócio novo que herda contato, empresa e indicador). Quem ficou em `pt_mapeado` tem ação **em lote**, com caixinhas marcadas por padrão: "Fechar os 14 que não saíram de Mapeado". É exceção explícita à regra do "nada em massa", porque aqui não é silenciar alarme, é fechar um ciclo que acabou de verdade numa data conhecida.

### 6.3 DETALHE DO NEGÓCIO

Duas colunas no desktop (`1fr 340px`), empilhado abaixo de 900px. Ordem no celular: faixa, stepper, ações, próximo passo, barra de registrar, timeline, contato, indicação, saúde, valores, observação.

```
‹ Funil / Patrocínio
SHINERAY DO BRASIL                        [PATROCÍNIO] [MOTO 2027]
Cota master
R$ 180.000        60% no estágio · R$ 108.000 ponderado
pediu 220.000, está em 180.000
●───●───●───○───○   Em aprovação, há 11 dias
[ Registrar ]  [✓ Ganho]  [✕ Perdido]  [❄ Congelar]
⚠ PRÓXIMO PASSO  [Cobrar aprovação interna ▾] [22/09] [ok]
┌ REGISTRAR ───────────────────────────────────────────────┐
│ [☎][💬][👥][📄][✉][📝]                                   │
│ [ o que aconteceu...                                   ] │
│ aconteceu em [09/09]   ☑ marcar próximo passo            │
│ [Cobrar resposta ▾] [+2d][+1sem][+15d][data]   [Salvar]  │
└──────────────────────────────────────────────────────────┘
▌LINHA DO TEMPO   (últimas 10, [ver mais])
```

Lateral: **Quem decide** (nome, cargo, botões WhatsApp e ligar), **Indicação** ("via João Bregantim, 3 indicações, 1 ganho" com o telefone dele ao lado, porque é a quem se pede um empurrão quando trava), **Saúde** (dias no estágio, dias sem contato, previsão, dias no funil), **Outros negócios da mesma empresa**, **Link da proposta**.

A barra de registrar fica **acima** da timeline: quem abre o negócio geralmente abre pra registrar, não pra ler. No celular ela substitui a navegação de baixo, e é reposicionada por `window.visualViewport` quando o teclado sobe, senão no iOS ela fica atrás do teclado e o Salvar some, que é exatamente a tela que decide a adoção.

**Observação fixa (`obs`)** no topo do bloco de valores: "não ligar antes das 10h", "o contrato passa pelo jurídico em Curitiba". É a memória que o vendedor usa de verdade e que não cabe numa atividade que afunda na timeline.

### 6.4 REGISTRO DE ATIVIDADE, o fluxo mais usado

Alvo: **2 cliques até salvo**, 1 se o tipo repete (o tipo vem pré-selecionado com o último usado na sessão). `Ctrl+Enter` salva.

Tipos: Ligação, WhatsApp, Reunião, Proposta enviada, E-mail, Nota. Não existe tipo "Tarefa": tarefa não aconteceu, vai acontecer, e isso já é o próximo passo.

O toggle "marcar próximo passo" vem **ligado**, com tipo e data sugeridos pelo estágio. A obrigação é **picklist mais data**; o texto livre é opcional. Picklist resolve o problema que data sozinha criava (vira botão de adiar alarme) sem cair no campo de texto obrigatório que gera "TBD" e "ver depois".

Os chips de data **espalham**: `+2d` sorteia 2 ou 3 dias úteis, `+1sem` de 6 a 9, `+15d` de 12 a 18. Sem isso, uma reancoragem de 6 negócios com um clique em `+1sem` empilha os 6 no mesmo dia, e a fila que existe pra parecer terminável vira a parede.

**Ao salvar:** 1 escrita em `crm_atividades`, 1 update no negócio (`ultimoToqueEm`, `ultimoToqueTxt`, `proxTipo`, `proxData`, `vezesAdiado` se for o caso), **0 leituras**. O item entra na timeline com opacidade 0.6 e um reloginho, e vira opaco **quando a promise do `set` resolver**, não antes. Se falhar, vai pro `crm_outbox` e ganha o selo "não enviado", com contador no cabeçalho. Toast só depois do servidor confirmar. Sem isso o CRM diz "salvo" antes de ter salvo, e esse é o momento exato em que um vendedor para de usar CRM.

Se o tipo foi **Proposta enviada** e o estágio ainda não é o de proposta, o toast ganha a ação "Mover pra Proposta?". Sugere, nunca move sozinho.

### 6.5 WhatsApp, que é onde a venda acontece

```js
function linkWhats(tel, texto){
  var d = String(tel||"").replace(/\D/g,"");
  if(d.length < 10) return null;                       // sem DDD nao monta
  if(d.indexOf("55") !== 0) d = "55" + d;
  return "https://wa.me/" + d + (texto ? "?text=" + encodeURIComponent(texto) : "");
}
```

Botão em todo cartão, linha de fila, tarefa e contato. Ao lado, `tel:` pra fixo com ramal, e `mailto:` quando só houver e-mail (que é o canal real da montadora).

**Captura no retorno**, que é o truque central: ao clicar em abrir conversa, grava `crm_pendente = {negocioId, quando}` **no `localStorage`**, não em memória (no celular o webview é descarregado enquanto o WhatsApp está aberto, e a memória some junto). Na volta ao foco **e também na inicialização**, se existe pendente com menos de 3 horas, aparece uma barra no rodapé com uma pergunta:

```
Falou com o Rodrigo?   [ Falei ]  [ Sem resposta ]  [ Depois ]
```

"Falei" e "Sem resposta" abrem direto o bloco de próximo passo. O registro acontece **na volta**, com a conversa fresca, e não antes de sair, quando a pessoa está com pressa.

### 6.6 BUSCA

Obrigatória. Sem ela, quando o telefone toca e é "o Rodrigo da Shineray", o vendedor tem 4 segundos e o sistema não responde, porque o negócio pode estar no balde futuro, que por definição não aparece em tela nenhuma.

Roda **em memória**, sobre os ~250 registros já carregados, zero leitura. Normaliza (minúsculo, sem acento) e casa por **prefixo de palavra**, não substring solta: `"mar"` acha "**Mar**roca" e "Ana **Mar**ia", não acha "Ca**mar**go". Também casa telefone por dígitos. Resultado em três grupos, negócios primeiro (no comercial se digita o nome da empresa mas o que se quer abrir é o negócio dela), máximo 5 por grupo, com desambiguador na segunda linha.

**Busca vazia vira cadastro:** `[+ Criar negócio pra "padaria do zé"]` com o nome já digitado. Quando o vendedor busca alguém que não existe, é porque acabou de receber uma indicação.

### 6.7 CADASTRO

**Negócio: 2 textos e 3 toques.** Empresa, tipo (2 botões), origem (chips), quem indicou se for indicação, próximo passo (tipo mais data). Menos de 20 segundos. O título é gerado sozinho (empresa mais tipo mais ano) e é editável.

- **Quem indicou é campo de busca que aceita nome novo**: digitou "Carla Menezes" e não achou, o próprio campo oferece "criar contato Carla Menezes" e grava o `indicadorId` na hora, só com o nome. Sem isso o vendedor marca "não veio de indicação" pra conseguir salvar, e o dado de primeira classe da casa nasce mentindo em metade da base.
- **`prevFechamento` nunca fica vazia**: nasce em `hoje + ciclo mediano do tipo` (34 dias agência, 96 patrocínio) com `prevEstimada:true`, mostrada em cinza como "data estimada". Negócio com data estimada **não entra em alta chance**. Isso mata o campo obrigatório que o vendedor odeia e mata o `undefined` que quebra a tela.
- **Empresa duplicada:** ao digitar, se `empresaChave` casar por prefixo com outra existente, aparece "essa empresa já tem 2 negócios" com o botão de usar a grafia existente. Com duas pessoas digitando, "Honda", "Honda do Brasil" e "HONDA MOTOS" viram três empresas em três meses.

**Contato: nome mais uma forma de contato.** Nada mais.

### 6.8 ESTADOS, ERRO E OFFLINE

Nada de `confirm()` do navegador. Folha de baixo pra tudo com consequência, e **Desfazer de 8 segundos** em vez de confirmação dupla: mover errado acontece toda semana, confirmar toda vez cobra um clique de todo mundo pra proteger de um erro de 8 segundos. Marcar Perdido e Ganho têm folha própria (Perdido exige o motivo em picklist).

Cinco estados vazios, com cinco textos diferentes: carregando; vazio de verdade (com botão de cadastrar); vazio por filtro (com limpar filtros); **vazio bom** (fundo `--lime-dim`, borda `--lime-border`: "Nada atrasado. Funil em dia."); e erro. Repetir o texto de primeiro uso numa lista filtrada faz a pessoa achar que perdeu os dados.

**Pontinho de sync** no cabeçalho, três estados, mais **faixa** que não some sozinha:
- Sem conexão: "Sem conexão. O que você registrar fica guardado no aparelho e sobe sozinho quando voltar." Com o contador de pendentes.
- `resource-exhausted`: "O limite diário do banco foi atingido. Nada se perdeu, mas nada novo sobe até amanhã. Fale com o Eric."
- `permission-denied`: "A gravação foi recusada pela regra do banco (o formato do documento mudou). A tela mostra a última versão salva."

As três mensagens são diferentes de propósito. Em 18/08/2026 todo mundo achou que era sinal ruim enquanto era cota estourada.

**Rascunho local:** enquanto o campo de atividade tem texto, grava em `crm_rascunho` a cada 3 segundos. O iOS mata o Safari em segundo plano o tempo todo.

**`localStorage` com cinto:** todo `getItem` e `setItem` dentro de `try/catch`. No `QuotaExceededError`, apaga o cache de timeline e tenta de novo uma vez; se falhar, roda sem cache e avisa no cabeçalho. O cache é conveniência e nunca pode ser caminho de falha. Todas as chaves com prefixo `crm_`, porque o `contato-lab.github.io` divide origem com os outros painéis da casa.

### 6.9 IDENTIDADE E CELULAR

```css
:root{
  --lime:#C8FF00; --lime-dim:rgba(200,255,0,0.10); --lime-border:rgba(200,255,0,0.25);
  --bg:#0A0A0A; --card:#141414; --card2:#1A1A1A;
  --text:#FFFFFF; --text2:#CCCCCC; --text3:#888888;
  --green:#00FF88; --red:#FF4444; --warn:#FFB800;
  --border:rgba(255,255,255,0.07); --radius:14px; --rad-sm:8px;
  --display:'Barlow Condensed',sans-serif; --body:'Inter',sans-serif;
}
```

Barlow Condensed 700/900 em título e número grande, Inter no corpo. Botão primário `--lime` com texto `#0A0A0A`. Sem biblioteca de gráfico: todo gráfico do Painel é barra horizontal em CSS (`.bartrack` mais `.barfill`), igual ao Cezinha.

Abaixo de 820px, `input, select, textarea { font-size:16px !important }`, senão o iOS dá zoom no foco e a tela desalinha. Alvo de toque mínimo 44px. Ação principal no terço inferior. Nenhuma tabela com rolagem horizontal: no celular, tabela vira lista de cartões, e a matriz de previsão vira três cartões empilhados.

---

## 7. AS FÓRMULAS DE CADA NÚMERO

Todos calculados em memória sobre o array já carregado. Trocar filtro, alternar mês e edição, ordenar e abrir qualquer bloco custa **0 leitura**.

**Regra de amostra pequena, vale pra todo número derivado:**
- `n >= 8`: mostra a estatística normal.
- `n` de 3 a 7: mostra a **mediana**, escreve `n=5` ao lado em `--text3`, opacidade 70%.
- `n < 3`: **não mostra média nenhuma**, lista os valores. Com 2 fechados, "ticket médio R$ 41.000" é pior que "R$ 62.000 e R$ 20.000".

**Todo número grande do Painel é clicável e abre a lista dos negócios que entram na conta.** Número que não pode ser aberto é número em que ninguém acredita, e um número desacreditado derruba o sistema inteiro junto.

### 7.1 Carteira e movimento do mês

```js
// CARTEIRA ATIVA (R$/mes). Contrato recorrente nao termina por data, termina
// por evento: quem filtra por vigencia apaga o cliente que continua pagando.
carteira      = Σ mrrDe(n)  para tipo=="agencia" && status=="ganho" && !n.encerradoEm
carteiraFee   = Σ n.ag.mrr  (mesma base)                     // linha 1, dinheiro contratado
carteiraVerba = carteira - carteiraFee                        // linha 2, estimado sobre verba
novoMrr(M)    = Σ mrrDe(n)  para status=="ganho" && mes(n.fechadoEm)==M && tipo=="agencia"
mrrPerdido(M) = Σ mrrDe(n)  para status=="encerrado" && mes(n.encerradoEm)==M
                             && n.motivoEncerramento != "renovado"
mrrLiquido(M) = novoMrr(M) - mrrPerdido(M)
fechadoMes(M) = Σ valorRefDe(n) para status=="ganho" && mes(n.fechadoEm)==M
verbaSobGestao= Σ n.ag.verbaMidia (carteira ativa)            // linha propria, "nao e receita"
```

`mrrLiquido` é o número que um dono de agência olha primeiro e quase nenhum CRM mostra: fechar R$ 3.500 e perder R$ 4.000 no mesmo mês é encolher, e sem a linha de perdido o CRM comemora o encolhimento. `mrrPerdido` aparece quebrado em duas linhas na tela, cancelamento e não renovação, porque as correções são opostas.

### 7.2 Previsão do mês

Soma **valor cheio** em três linhas, nunca um ponderado como manchete. Com 10 a 15 negócios abertos o resultado de cada um é binário: ou entra R$ 80.000 ou entra R$ 0, nunca R$ 56.000.

```js
function altaChance(n){
  if(n.status !== "aberto") return false;
  if(n.prevEstimada) return false;                                  // data automatica nao conta
  var f = FUNIL[n.tipo], i = idxEstagio(n), iC = f.findIndex(e=>e.compromisso);
  var planoVivo = n.proxData >= hojeISO() && diasEntre(hojeISO(), n.proxData) <= 21;
  return i >= iC                                                    // 1. comprador se comprometeu
      && mes(n.prevFechamento) === mesRef                           // 2. previsto pra este mes
      && n.prevFechamento >= hojeISO()                              // 3. a data nao venceu
      && (planoVivo || diasVencido(n.ultimoToqueEm) <= 10);         // 4. plano vivo OU toque recente
}
PISO(M)     = Σ valorRefDe(n) para status=="ganho" && mes(n.fechadoEm)==M
PROVAVEL(M) = PISO(M) + Σ valorRefDe(n) para altaChance(n)
LIMPO(M)    = PROVAVEL(M) menos os de altaChance com riscoAlto(n)
```

O teste 4 é "plano vivo **ou** toque recente", não só "toque nos últimos 10 dias". Quem marcou reunião pro dia 25 continua em alta chance até o dia 25; quem passou do próprio prazo cai. A versão original ensinava o vendedor a cutucar o registro de nove em nove dias, e aí `ultimoToqueEm` deixa de significar qualquer coisa e três blocos mentem juntos.

**Duas réguas simultâneas, nunca no mesmo número:**

```
Setembro 2026                          AGÊNCIA      PATROCÍNIO       TOTAL
  Fechado                             R$  24.000    R$  22.200   R$  46.200
  Alta chance                         R$  24.000    R$  60.000   R$  84.000
  ------------------------------------------------------------------------
  Piso R$ 46.200 · Provável R$ 130.200 · Sem risco alto R$ 116.400
  Meta R$ 120.000, faltam R$ 3.350
  Novo MRR em jogo: R$ 2.000/mês fechado, R$ 2.000/mês em alta chance
```

Nota fixa embaixo da coluna Total, nunca em tooltip: "Soma valor total de contrato (fee vezes meses) com cota de patrocínio (pagamento único). Serve pra dimensionar o esforço comercial. Não é caixa do mês."

A distância entre Provável e Sem risco alto é a métrica mais honesta do sistema: mede quanto da previsão está apoiado em negócio que ninguém toca há semanas.

**Patrocínio tem alternador MÊS | EDIÇÃO**, porque cota não obedece mês, obedece ciclo de verba da montadora. Mês vazio de patrocínio não some nem parece problema: mostra "Nenhuma cota prevista pra setembro. Na edição moto 2027: R$ 118.000 em aberto". E `prevFechamento` de patrocínio **não é a data do evento**: nasce sugerida em `corteComercial` da edição, e é editável.

### 7.3 Saúde do funil

```js
emDia(n) = n.proxData >= hojeISO()
        && diasVencido(n.ultimoToqueEm) <= TOQUE[n.tipo].normal
tocados7 = abertos.filter(n => diasVencido(n.ultimoToqueEm) <= 7).length
```

- **"9 de 12 negócios em dia"**, com barra. Não virou índice de saúde de 0 a 100 de propósito: índice composto é número que ninguém sabe o que fazer com; "3 negócios fora" tem nome, valor e telefone.
- **Faixa de confiança no topo do Painel: cobertura de toque**, `tocados7 / abertos`. Verde acima de 70%, `--warn` de 40 a 70, `--red` abaixo. Medir só o toque mais recente da base inteira dá verde permanente com um vendedor que cuida do negócio grande e deixa os outros onze apodrecerem.
- **Concentração:** `maior / total dos abertos do mês`. Só aparece com 3 ou mais abertos. Acima de 40%, `--warn`; acima de 60%, `--red` com "A previsão do mês é uma moeda girando num negócio só". Quatro linhas de código e é o alerta mais honesto do painel.
- **Vencidos:** qualquer quantidade já é `--warn`; algum vencido há mais de 30 dias é `--red`, porque aí não é atraso, é negócio morto contando na conta.
- **Cobertura de pipeline (pipeline dividido pela meta) fica fora.** Comparar pipeline do mês com meta do mês acende vermelho na última semana de todo mês por construção, e na terceira vez o gestor para de olhar o bloco inteiro.

### 7.4 Ganhos e perdas (janela de 12 meses)

```js
// conversao(i): so negocios JA FECHADOS entram. Aberto parado no estagio 2
// ainda pode avancar; conta-lo derruba a conversao e o diagnostico sai invertido.
base(i)   = fechados.filter(n => n.tipo==f && n.maiorEstagio >= i)
passou(i) = base(i).filter(n => n.maiorEstagio >= i+1)
conversao(i) = passou(i).length / base(i).length          // sempre com o n ao lado

perdasReais   = perdidos.filter(n => n.motivoPerda != "fora_do_perfil")
taxaGanho     = ganhos.length / (ganhos.length + perdasReais.length)   // a manchete
cicloMediano  = mediana( ganhos.map(n => diasEntre(n.criadoEm, n.fechadoEm)) )
cicloAtePerder= mediana( perdasReais.map(...) )           // segunda linha, e informacao
```

"Fora do perfil" sai do denominador: contar desqualificação como derrota pune a única disciplina que faz um time de duas pessoas sobreviver, que é dizer não.

**Motivo de perda ranqueado por R$ perdido, não por quantidade**, com as duas colunas visíveis. Três contratos de R$ 3.500/mês perdidos por preço somam menos que uma cota perdida por verba, e ordenar por quantidade esconde exatamente o problema que importa.

`mrrInicial` e `pt.valorInicial` (gravados uma vez, nunca sobrescritos) fazem a análise de preço funcionar: sem eles, o desconto dado antes de perder apaga o valor proposto, e a conclusão sai invertida justamente nos casos que importam. O card mostra "pediu 220.000, está em 180.000" quando os dois diferem, que é também o que o dono quer ver antes de aprovar desconto.

**Ticket:** agência mostra três números com três unidades (ticket de contrato em R$, fee mediano em R$/mês, duração mediana em meses), porque R$ 42.000 pode ser 3.500 por 12 ou 7.000 por 6, e são negócios diferentes. Patrocínio mostra cota mediana por edição.

### 7.5 Indicação

Agrupado por `indicadorId` sobre os negócios em memória, ordenado por R$ gerado.

| Quem indicou | Indicações | Viraram cliente | Conversão | Gerado | Recorrente | Em aberto |
|---|---|---|---|---|---|---|
| Carla Menezes | 4 | 3 | 75% (3 de 4) | R$ 96.000 | R$ 4.500/mês | R$ 24.000 |

- Taxa sempre com o `n` ao lado. 100% com uma indicação não é 100%, é uma indicação.
- MRR gerado e cota gerada em colunas separadas, nunca somadas.
- **Valor médio da indicação** = R$ gerado nos ganhos dividido por **todas** as indicações recebidas, não só as ganhas. E a linha declara o buraco: "(3 negócios com origem indicação e indicador não informado ficaram de fora)".
- **Indicador esfriando**: duas ou mais indicações ganhas e mais de 120 dias sem indicar. É a ação de maior retorno disponível pra uma agência que vive de indicação, e custa cinco linhas.
- **Indicado contra não indicado**: taxa de ganho, ciclo mediano e ticket mediano, lado a lado, com a linha montada com os números reais ("Negócio indicado ganha 2,8 vezes mais e fecha 33 dias antes"). Se um dos lados tem `n < 3`, a linha some.

Indicação **não muda** estágio de entrada nem probabilidade. Muda quatro coisas: prazo de 1 dia útil no primeiro estágio, prioridade no desempate da fila, contexto na tela com o telefone de quem indicou, e a tarefa automática de avisar o resultado.

### 7.6 Cobertura da edição (patrocínio)

```js
vendido  = Σ pt.valor dos ganhos da edicao
emAberto = Σ pt.valor dos abertos da edicao
diasAteEvento  = diasEntre(hojeISO(), EDICOES[k].dataEvento)
diasAteOCorte  = diasEntre(hojeISO(), EDICOES[k].corteComercial)
```

Mostrado como "Moto 2027: R$ 420.000 vendidos de R$ 600.000 de meta, R$ 118.000 em aberto, faltam 48 dias pro corte comercial". É a única métrica que junta dinheiro e tempo, e é a pergunta que o dono faz de verdade.

---

## 8. MODELOS DE MENSAGEM

Vivem num objeto JS no código. O sistema troca `{chave}` pelo que sabe, e **a prévia é editável antes de enviar**: o que faltar vira um campo inline na própria prévia, preenchido na hora. O botão de abrir conversa **crua** (sem modelo) nunca é bloqueado. Sem isso, um modelo que pede um campo que não existe trava pra sempre justamente o negócio de maior ticket.

Variáveis: `{nome}` `{empresa}` `{indicador}` `{vendedor}` `{mrr}` `{meses}` `{cota}` `{valor}` `{evento}` `{dataEvento}` `{entregaveis}` `{corte}` `{diaRetorno}` `{assunto}` `{gancho}` `{hora}`.

**M1, primeiro contato por indicação**
> Oi {nome}, aqui é {vendedor} da Agência Lime. O {indicador} falou de vocês e pediu pra eu te procurar. A gente cuida de mídia e tráfego pago, e o que eu faço primeiro é entender onde está o gargalo antes de propor qualquer coisa. Consegue 15 minutos essa semana? Pode ser quarta às 10h ou quinta às 16h, o que for melhor pra você.

Duas datas concretas em vez de "quando você puder", que transfere o trabalho pro outro lado e por isso não é respondido.

**M2, proposta enviada (agência)**
> {nome}, acabei de mandar a proposta no seu e-mail. Resumo em três linhas: {mrr} por mês de gestão, contrato de {meses} meses, e a verba de mídia entra separada, você define o valor. Se tiver algum ponto que não fechou eu ajusto. Te ligo {diaRetorno} pra gente falar sobre?

Sem cláusula sobre comissão de verba: política comercial escrita dentro de software vira promessa por escrito no WhatsApp do cliente, e corrigir exige subir arquivo novo.

**M2b, proposta de cota (patrocínio)**
> {nome}, segue a cota {cota} do {evento}, que acontece em {dataEvento}. Valor de {valor}. O que entra: {entregaveis}. As cotas são por ordem de fechamento e a grade de material fecha em {corte}, então se fizer sentido pra vocês eu seguro essa até lá. Qualquer ajuste no pacote a gente conversa.

**M3, cobrança com saída (10 a 14 dias sem resposta)**
> {nome}, não vou ficar insistindo à toa. Me diz só uma coisa: fica pra frente ou eu arquivo por enquanto? Qualquer uma das duas está tranquilo, só quero saber pra não te encher.

Dar a saída é o que faz a pessoa responder. Resposta negativa vale mais que silêncio, porque libera a fila.

**M4, reativação (60 dias ou mais)**
> {nome}, faz um tempo que a gente falou sobre {assunto}. Lembrei de vocês agora por causa de {gancho}. Se quiser eu retomo de onde parou, está tudo aqui comigo. E se o momento não for esse, sem problema nenhum.

`{gancho}` é o único campo que o vendedor digita na hora, de propósito. Reativação sem motivo real é spam e queima o contato.

**M5, pedido de indicação**
> {indicador}, pergunta rápida. Tem alguém que você conhece que está precisando resolver marketing ou mídia e que faria sentido eu conversar? Não precisa apresentar nem nada, só me passa o nome que eu procuro e falo que veio de você.

**M6, agradecimento a quem indicou** (a tarefa criada no fechamento)
> {indicador}, fechamos com a {empresa}. Valeu demais pela indicação, foi você que abriu essa porta. Qualquer coisa que eu puder fazer do seu lado é só chamar.

**M7, véspera de reunião**
> {nome}, confirmando nossa conversa de amanhã às {hora}. Vou levar {assunto}. Se precisar remarcar me avisa hoje que eu encaixo.

**Resumo colável do dia e do mês**, botão no topo, `navigator.clipboard.writeText` com a string montada **antes** da chamada (no Safari do iPhone, qualquer `await` entre o clique e o `writeText` faz a permissão ser recusada em silêncio e o botão parecer quebrado). Fallback: `<textarea>` num modal com o texto já selecionado.

```
*LIME | Comercial, 10/09/2026*

Fechado no mes: R$ 46.200
Meta do mes: R$ 120.000 (38% atingido)
Alta chance: R$ 84.000 em 2 negocios
Base recorrente ativa: R$ 18.500/mes (variacao no mes: +R$ 2.100/mes)

*Vai fechar*
1. Shineray, cota master moto, R$ 180.000, previsto 18/09/2026
2. Marroca, agencia, R$ 3.500/mes por 12 meses, previsto 22/09/2026

*Travado*
3 negocios sem contato ha mais de 7 dias, R$ 51.000 no total
1 negocio com data vencida: Mundo Brinca, venceu ha 12 dias

*Indicacao*
Quem mais trouxe: Carla Menezes, 4 indicacoes, R$ 96.000 gerados
```

**Lembrete não sai do CRM, sai da agenda do celular.** No primeiro login, um passo único cria um evento recorrente no Google Agenda (`calendar.google.com/calendar/render?action=TEMPLATE&recur=RRULE:FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR`), 8h45, com o link do CRM na descrição, mais o `.ics` pra quem usa Apple Calendar. Um arquivo estático não tem como acordar ninguém, e fingir que tem é como o projeto morre na terceira semana. O passo **não é marcado como concluído por clique**: vira uma linha discreta que volta pro topo depois de 3 dias sem abrir. Todo próximo passo com data a mais de 10 dias e toda reunião com hora ganham o botão "pôr na agenda" (evento único).

---

## 9. ORÇAMENTO DE LEITURA DO FIRESTORE

**Volume assumido no ano 1:** 40 negócios abertos, 90 fechados, 120 contatos, 10 tarefas abertas, 2.000 atividades acumuladas, 4 acessos.

| Ação | Leituras |
|---|---|
| Login (só quando a sessão de 7 dias expira) | 4 (`crm_acessos`) |
| **Carga fria** (aparelho novo, cache limpo, Safari que descartou storage) | 130 + 120 + 10 + 1 = **261** |
| **Abertura com cache quente** | **4** (3 consultas delta que voltam vazias + `crm_config/geral`) |
| Abertura depois de um dia de trabalho do outro usuário (20 documentos mexidos) | 24 |
| Abrir um negócio nunca aberto (timeline `limit(10)`) | 10 |
| Reabrir o mesmo negócio sem alteração (cache por `at`) | 0 |
| "Ver mais" na timeline | 20 |
| Trocar de aba, filtrar, buscar, ordenar, abrir qualquer relatório | **0** |
| Registrar atividade, mover estágio, adiar, congelar, cadastrar | **0** (2 a 3 escritas) |
| Importar 30 negócios colados da planilha | **0** (1 lote, 30 escritas) |
| Re-sync ao voltar o foco (teto de 1 a cada 30 min e 8 por dia) | 4 a 12 cada |

**Dia típico do vendedor:** 5 aberturas quentes (20) + 6 re-syncs de foco (~40) + 5 timelines novas (50) = **110 leituras**.
**Dia típico do gestor:** 2 aberturas (8) + 2 re-syncs (10) + 6 timelines (60) = **78 leituras**.
**Os dois juntos: cerca de 190 leituras por dia.** Somando uma carga fria por semana amortizada (261 ÷ 7 ≈ 37), **cerca de 225 leituras por dia, 0,45% do teto gratuito de 50.000 que é compartilhado com a Central do Evento.**

**Pior caso realista** (cinco limpezas de cache no mesmo dia, dois aparelhos novos, dia de muita consulta): 261 × 5 + 400 ≈ **1.700 leituras, 3,4% da cota**. A Central continua com 96% dela.

**O que o desenho ingênuo custaria:** `onSnapshot` em `crm_negocios`, `crm_contatos` e `crm_atividades` cobra o tamanho da coleção **a cada vez que a escuta liga**, e ela religa em toda recarga de página. São 2.250 documentos por abertura. Com dois usuários e dez aberturas cada, passa de 45.000 por dia, que é a cota inteira. É a mesma conta que derrubou a Central em 18/08/2026.

**As sete regras que sustentam esse número**
1. Zero `onSnapshot`, em documento ou em coleção. Tudo `get()`.
2. Sincronização incremental por `at`, com margem de 5 minutos, e `at` sempre `serverTimestamp` (celular adiantado cegaria o delta em silêncio e o cache mentiria sem nenhum sinal na tela).
3. Atividade só é lida ao abrir o negócio, com `limit(10)`, cacheada e invalidada pelo `at` do negócio.
4. Re-sync no foco com teto duro: 1 a cada 30 minutos, 8 por dia por aparelho, gravados no `localStorage`. Sem teto, quem alterna entre o CRM e o gerenciador de anúncios 40 vezes por dia paga 40 syncs.
5. Todo relatório, semáforo, ranking, previsão e busca é aritmética local sobre o array já carregado.
6. Nenhum contador agregado gravado. Ranking de indicação e métricas saem de memória, então não existe documento de resumo pra desalinhar nem pra recalcular.
7. **Contador de leituras da sessão** gravado no `localStorage` e mostrado no painel do Admin. Sem telemetria, a descoberta de que estourou a cota acontece do mesmo jeito que em 18/08/2026: com a produção caída.

**Escritas por ação** (que ninguém contou nas propostas originais): criar negócio = 2 (negócio + atividade de criação). Registrar atividade = 2. Mover estágio = 2. Fechar com indicação = 3 (negócio, atividade, tarefa de avisar). Nenhuma ação usa `runTransaction`: transação obriga leitura no servidor e trava a gravação justamente quando a cota de leitura acaba, que foi o que travou a Central.

---

## 10. O QUE FICA DE FORA DA VERSÃO 1

| Cortado | Por quê | Onde foi parar |
|---|---|---|
| Documento-índice único com todos os negócios | Uma requisição apaga a base; toda edição reescreve 100 KB; duas abas se sobrescrevem | Coleção com delta por `at` |
| Empresa como entidade própria | Entidade só paga com vários contatos e vários donos na mesma conta. Aqui seria mais uma tela pra manter e mais um lugar pro dado divergir | Campo texto com `empresaChave` normalizada, autocomplete e aviso de duplicata |
| Entidade "indicador" separada de contato | O Paulo da Shineray existiria duas vezes e nunca mais bateria | `ehIndicador` no contato |
| Contadores de indicação gravados (`nGanhos`, `valorGerado`) | Máquina de incremento atômico, lote, desalinhamento e botão de recalcular, tudo pra evitar uma conta que roda de graça em memória | Calculado no render |
| Coleção de auditoria e coleção de log | Cresceria pra sempre, ninguém leria, e pra responder "quem mudou o estágio" exigiria uma consulta por negócio | A timeline do negócio é o log |
| Kanban arrastável | Consome a maior parte do tempo de desenvolvimento (toque, ordenação, estado intermediário) e entrega zero informação nova. Long press no iOS ainda dispara sozinho durante a rolagem | Folha de estágios, 2 toques, igual em tudo |
| Aba Atividades (feed global) | Com duas pessoas, ninguém abre um feed pra descobrir o que a outra fez: a outra está sentada do lado | Timeline por negócio e o bloco do Hoje |
| Lead scoring | Volume baixo, ticket alto, origem indicação. Pontuar 30 negócios é teatro | Alerta de esfriamento, que é sobre tempo |
| Probabilidade editável e forecast category (Committed / Best Case) | Só funciona com cultura de commit e consequência. Com um vendedor, é ele conversando com ele mesmo | Probabilidade sai do estágio; `riscoAlto` é derivado de comportamento observável |
| Ponderado calibrado pelo histórico | 40 linhas, tabela provisória e segunda varredura pra produzir um número que o próprio autor diz que ninguém vai olhar | A comparação Provável contra Sem risco alto |
| Cobertura de pipeline (3x a meta) | Acende vermelho na última semana de todo mês por construção, e na terceira vez o gestor para de olhar o bloco | Em aberto por funil, com meta |
| Campos MEDDIC | Cinco campos de texto longo matam a adoção na segunda semana | Critério de saída escrito na tela, mais `quemDecide` e `faixaVerba` em picklist |
| Entregáveis com checkbox e prazo de produção por item | É controle de entrega pós-venda, não de venda. Quem executa estande é a produção do festival, dentro da Central | Texto de uma linha por item na proposta, mais o corte comercial na tabela de edições |
| Colar conversa exportada do WhatsApp e vCard | Exportar conversa no celular são 6 toques e o resultado chega como anexo, não como texto colável. Ninguém faz duas vezes | Nada |
| Service worker, PWA com push, Web Share Target | Push exige servidor; Periodic Background Sync não existe no iOS; e o SW ainda serve `index.html` velho depois do upload pelo navegador, que é justamente o deploy da casa | Evento recorrente no Google Agenda |
| Notificação do navegador como promessa | Notificação que só aparece pra quem já está com a aba aberta é decoração | Título da aba com o contador |
| Cartão PNG pro slide (SVG mais canvas mais canShare) | Terceiro caminho de export, e o `@media print` já dá PDF vetorial com qualidade melhor | Texto colável mais impressão em PDF |
| Chart.js e html2canvas | 200 KB cada pra desenhar 12 retângulos | Barra em CSS |
| Anexo de arquivo | Exige Storage, cota nova, regra nova | `linkProposta`, texto |
| Importação de CSV com detecção de duplicado e seletor de 9 colunas | Código pra ser usado uma vez na vida, numa lista de 30 linhas que o vendedor confere no olho | Colar TSV com ordem fixa e prévia de 3 linhas |
| Perfil "comercial do festival" com funil restrito | Custa índice, segundo caminho de consulta e três vazamentos não resolvidos (atividade, contato, tarefa), pra uma pessoa que não foi contratada | A chave `funis` entra no dia em que a pessoa existir |
| Bloqueio progressivo por senha errada | Não segura script (quem ataca fala direto com o Firestore) e castiga o vendedor de dedo grande no celular | Botão Mostrar senha |
| Campo de dica da senha no banco | Pista de senha guardada em banco público é a senha, só que escrita devagar | Nada |
| Gamificação (dias seguidos, badge, streak) | Irrita adulto e queima a credibilidade da ferramenta. São duas pessoas na mesma sala | Nada |
| Metas por vendedor, SLA, rodízio, aprovação em níveis | Duas pessoas. O único prazo que sobrevive é 1 dia útil pra indicação, e sobrevive porque qualquer um entende o motivo | Nada |
| Tema claro | Identidade da Lime é escura e alternância dobra o teste visual | `@media print` inverte pro branco só na impressão |
| Rotação e arquivamento automático de negócio antigo | Necessário só depois de 2032 pela conta do próprio desenho | Nada |

---

## 11. INVARIANTES (o que o código nunca pode fazer)

1. Nunca `onSnapshot`, em coleção ou em documento.
2. Nunca `runTransaction`.
3. Nunca `delete` no Firestore. Exclusão é `arquivado:true`.
4. Nunca gravar campo derivado, com uma exceção documentada: `valorRef`.
5. Nunca somar MRR de agência com valor de patrocínio no mesmo número, e nunca somar permuta com dinheiro.
6. Nunca mudar o `tipo` de um negócio depois de criado.
7. Nunca gravar `at` que não seja `serverTimestamp`.
8. Nunca chave com ponto dentro de `set` (`set` não interpreta caminho de campo; só `update` interpreta). O erro cria campos de nome literal na raiz e faz a regra recusar toda gravação futura.
9. Nunca mostrar toast de "salvo" antes de a promise do `set` resolver.
10. Nunca campo de texto livre obrigatório. Onde é obrigatório, é picklist ou data.
11. Todo acesso ao `localStorage` dentro de `try/catch`.
12. Toda escrita passa por `gravar()`, e coleção não listada no porteiro nasce fechada.

---

## 12. ORDEM DE IMPLEMENTAÇÃO E IMPLANTAÇÃO

**Semana 1, o que faz o sistema existir:** login com hash e senha de recuperação, camada de sincronismo (cache, delta, outbox, pontinho de sync), cadastro de negócio e contato, tela Hoje, Funil em leitura com a folha de estágios, detalhe do negócio, registro de atividade, botão de WhatsApp, busca.

**Semana 2:** Painel com os quatro números e a lista por trás de cada um, tarefas, Acessos, resumo colável, exportar JSON de tudo (backup), `@media print`.

**Depois, na ordem da dor:** ranking de indicação completo, entrada rápida colando da planilha, virada de edição, CSV.

**Checklist de implantação, com dono e data**
1. Criar o repo no contato-lab, subir `index.html`, ligar o GitHub Pages.
2. Habilitar o provedor **Anônimo** no Authentication do console.
3. Copiar as regras atuais da Central do console, colar o bloco do CRM antes do `match /{document=**}` final, publicar o arquivo inteiro. Conferir a Central logo depois.
4. **Trocar a senha de recuperação do código** e subir de novo. O Mac não tem credencial git, então é upload pelo Chrome, e o upload substitui o arquivo inteiro: sempre partir da versão que está no repo.
5. Criar o alerta de orçamento no Google Cloud do projeto `festival-interlagos-2026` (o crédito vence em 18/11/2026).
6. Criar os acessos reais no painel e anotar cada senha no gerenciador de senha.
7. Primeira carga: 40 minutos cadastrando **só os negócios abertos agora**. Não existe backlog de digitação: cliente antigo e negócio perdido entram quando forem tocados.
8. Baixar o JSON de backup na primeira sexta e repetir uma vez por mês.
