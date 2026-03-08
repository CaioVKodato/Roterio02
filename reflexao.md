# Bloco de Reflexão — Roteiro 02

Responda individualmente, com suas próprias palavras. Cada resposta deve ter no mínimo **4 linhas**, citar pelo menos **um conceito técnico** e, quando aplicável, referenciar o código que você escreveu.

---

## 1. Síntese

Qual dos 7 tipos de transparência você considera mais difícil de implementar corretamente em um sistema real? Justifique com um argumento técnico baseado nos exercícios realizados.

Acho que a mais chata de acertar na prática é a **transparência de relocação**. Na migração (Tarefa 3) a gente só troca de instância entre uma requisição e outra, e o Redis guarda o estado. Na relocação o recurso se move *enquanto* o cliente ainda está usando. Aí você precisa de buffer de mensagens, máquina de estados (CONNECTED → MIGRATING → RECONNECTING) e garantir que nada se perca no meio do caminho. Qualquer falha na reconexão ou no reenvio do buffer quebra a ilusão pro usuário. Por isso é bem mais difícil que as outras.

---

## 2. Trade-offs

Descreva um cenário concreto de um sistema que você conhece (app, site, jogo) em que esconder completamente a distribuição levaria a um sistema menos resiliente para o usuário final.

Um exemplo é um app de delivery: se a tela de “rastreando pedido” se comportar como se fosse tudo local (uma única chamada que pode travar ou falhar em silêncio), o usuário fica sem feedback quando a rede cai ou o backend do restaurante demora. Ele não sabe se deve esperar, tentar de novo ou desistir. Se o app deixar explícito que está “conectando ao restaurante” ou “aguardando resposta da rede”, com timeout e mensagem de erro clara, a pessoa entende o que está acontecendo e pode decidir. Esconder a distribuição demais aqui piora a experiência em vez de melhorar — o conceito de *fail fast* e feedback explícito ajuda nisso.

---

## 3. Conexão com Labs anteriores

Como o conceito de `async/await` explorado no Lab 02 se conecta com a decisão de quebrar a transparência conscientemente, vista na Tarefa 7?

No Lab anterior a gente viu que `async/await` deixa claro no código que aquela operação pode “pausar” (o event loop fica livre pra outras tarefas). Na Tarefa 7 o `bom_pattern.py` usa justamente uma função `async` com nome `fetch_user_remote` e retorno `Optional[dict]` — ou seja, o programador que chama já sabe que é chamada de rede e que pode falhar. Quebrar a transparência de propósito com `async` e tipos explícitos força quem usa a função a tratar falha e latência, em vez de fingir que é uma consulta local. Então o async/await do Lab 02 é uma ferramenta pra essa “transparência consciente”: você sinaliza no código que ali a distribuição existe.

---

## 4. GIL e multiprocessing

Explique com suas palavras por que a Tarefa 6 usa `multiprocessing` em vez de `threading`. O que é o GIL e por que ele interfere na demonstração de race conditions em Python?

O **GIL** (Global Interpreter Lock) é um lock no interpretador do CPython que faz com que só uma thread execute bytecode Python por vez no mesmo processo. Com threading, duas threads não rodam “de verdade” em paralelo naquele processo — então a race condition do saldo (ler, dormir, escrever) pode até não aparecer direito em alguns testes. Com **multiprocessing** cada processo tem seu próprio interpretador e seu próprio GIL, então os dois processos realmente concorrem pelo mesmo dado no Redis. Aí a condição de corrida fica visível e reproduzível, e fica parecido com o cenário real de vários servidores acessando o mesmo recurso. Por isso a Tarefa 6 usa processos e não threads.

---

## 5. Desafio técnico

Descreva uma dificuldade técnica encontrada durante o laboratório (incluindo o provisionamento do Redis Cloud), o processo de diagnóstico e a solução. Se não houve dificuldade, descreva o exercício mais interessante e explique por quê.

Tive que instalar o `python-dotenv` e o `redis` pra rodar os scripts — deu ModuleNotFoundError no `dotenv` ao executar a instancia_a. O diagnóstico foi direto: o traceback apontou o import. Resolvi com `pip install python-dotenv redis` (o pacote no pip é python-dotenv, mas no código a gente importa como dotenv). O exercício que mais curti foi o da Tarefa 6 com o lock distribuído no Redis: ver o saldo dar errado no `sem_concorrencia.py` e depois ficar certo no `com_concorrencia.py` com o `distributed_lock` usando SET NX EX deixa bem claro por que um Lock de thread não resolve em sistema distribuído.

---

## Tarefa 4 — Questões de relocação (discussão em dupla)

Registre no bloco abaixo as respostas discutidas sobre a Tarefa 4:

1. **Migração vs. Relocação:** Qual é a diferença prática? Por que relocação é tecnicamente mais difícil?

Na migração você troca de servidor *entre* requisições — uma instância cai, outra sobe e lê o estado do Redis. Na relocação o recurso se move *durante* o uso: a conexão (ex.: WebSocket) muda de endpoint com o cliente ainda “conectado”. Por isso relocação é mais difícil: tem que bufferizar mensagens, reconectar e reenviar sem o usuário perceber, e qualquer falha no meio quebra a continuidade.

2. **Buffer e exactly-once:** O buffer interno garante semântica de entrega *exactly-once*? O que poderia causar duplicação ou perda?

Não garante exactly-once. Se a reconexão falhar depois de enviar parte das mensagens do buffer, ao tentar de novo pode reenviar as que já foram — duplicação. Se o processo cair depois de bufferizar e antes de drenar, as mensagens do buffer se perdem. Falta um mecanismo de confirmação (ack) e idempotência/retry para chegar perto de exactly-once.

3. **Máquina de estados:** Por que modelar estados explicitamente em vez de uma flag booleana `is_relocating`?

Porque o fluxo não é só “está relocando ou não”. Tem pelo menos três fases: conectado, migrando (bufferizando), reconectando (abrindo nova conexão e drenando buffer). Uma flag booleana não diferencia “bufferizando” de “reconectando”, e fica mais difícil tratar erros em cada fase e garantir que as transições (ex.: só voltar a CONNECTED depois de drenar o buffer) estejam corretas.

4. **Sistema real:** Cite um sistema real em que transparência de relocação é requisito.

Kubernetes: quando um Pod é reschedulado (ex.: o nó fica indisponível), as conexões que estavam naquele Pod precisam ser redirecionadas ou reconectadas de forma que o cliente sofra o mínimo. Outro exemplo é live migration de VMs (ex.: Hyper-V, VMware): a VM muda de host físico enquanto continua rodando, e o usuário não deve perceber a mudança.
