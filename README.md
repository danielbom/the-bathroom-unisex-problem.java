# O problema do banheiro unisex - The Bathroom Unisex Problem

Projeto em java com a resolução do problema de sincronização banheiro unisex, apresentado como atividade do curso de Programação Concorrente na UTFPR.


## Descrição
***

O problema do banheiro unisex surgiu a partir de uma história de uma mulher que estava trabalhando em um local muito longe do banheiro feminino. O banheiro mais próximo era reservado apenas para homens. Então ela decidiu solicitar ao seu chefe que torna-se este banheiro um banheiro unisex. O chefe concordou com a proposta a fim de ajudá-la, mas impôs algumas restrições.

- O banheiro só pode conter ou homens ou mulheres, nunca ambos ao mesmo tempo.
- O banheiro deve ter um limite de 3 pessoas, para evitar que funcionários aproveitem para 'matar hora' com amigos.

A solução deste problema pode ser feita com diversas métodos de sincronização. Aqui, foram implementados uma versão com monitores e uma versão com semáforos, não identicas. A versão com monitores mantem um acordo de intercalação de gêneros, por exemplo, após um grupo de homens irem, caso haja mulheres, os próximos a irem serão mulheres. Já a versão com monitores o acordo é de justiça, isto é, a ordem de chegada de cada pessoa importa, logo, é possível ter grupos de mulheres seguido por outro grupo de mulheres, ou então, quando o grupo não satisfaz o limite permitido, ele é consumido de forma fragmentada. Vale lembrar que a versão com monitores não há justiça de ordem de chegada. Execute os testes de intercalação, o consecutivo e o proporção 2/1 para entender melhor.

## Codificação
***

Para resolver o problema, foi definido algumas classes básica, Pessoa (People), Sexo (Sex) e Interface Banheiro Unisex (IBathroomUnisex). As classes Pessoa e Sexo são utilizadas para abstrair o problema, onde Pessoa executa um simples função de usar o banheiro, e possui um Sexo atribuido, para ser analisado antes de entrar no banheiro. A Interface Banheiro Unisex é utilizada para definir os métodos comuns para se usar o banheiro, isto é, entrar no banheiro e sair do banheiro. Desta forma, cada classe que implementa esta interface deve implementar a lógica de entrar e a lógica de sair do banheiro, definindo as regras necessárias.

### Monitores

A implementação do problema com monitores foi mais natural para mim. Quando se usa monitores, tecnicamente podemos pensar no problema de forma direta e procedural, apenas criando variáveis de controle e as manipulando de forma direta. Contudo, o código ficou mais denso e maior. As variáveis de controle do monitor são:

```java
    int womans = 0; // Contador de mulheres usando o banheiro
    int mans = 0;   // Contador de homens usando o banheiro

    int womansWaiting = 0;  // Contador de mulheres esperando
    int mansWaiting = 0;    // Contador de homens esperando

    Sex priority = Sex.FEMALE;  // Sexo prioritário atual

    int consecutive = 0;  // Contador de pessoas consecutivas
    int maxConsecutive;   // Limite de pessoas consecutivas permitidas
```

Ao mudar a prioridade é necessário resetar o contador de pessoas consecutivas, e na constante necessidade de realizar esta operação, foi criado um método para fazer isto.

```java
    private void changePriority(Sex sex) {
        priority = sex;
        consecutive = 0;
    }
```

O método de entrar no banheiro deve ser atômico, alcançado utilizando o modificador *__syncronized__*. Ao fazer isto, não é possível garantir a utilização do banheiro por ordem de chegada, contudo o restante da implementação se torna mais simples. O método recebe a pessoa que esta querendo entrar no banheiro, para verificar seu sexo e então aplicar as regras necessárias.

Quando uma pessoa entra, é imediatamente incrementado o contador de espera para o seu sexo **(womansWaiting++, mansWaiting++)**. Após isso, fazemos a validação das regras que o problema impõe. Por exemplo, para uma mulher poder usar o banheiro é necessário que a prioridade seja feminina, a quantidade de mulheres seja menor que o limite de mulheres permitidas no banheiro e o numero de homens no banheiro deve ser 0, logo a regra de espera é a seguinte: **(priority == Sex.MALE || womans > limit || mans != 0)**. Caso não exista homens esperando, e o limite de mulheres no banheiro ainda não alcançou o limite, e nem existe homens usando o banheiro, a próxima mulher pode alterar a prioridade para feminino e sair do laço de espera, verificado em **(mansWaiting == 0 && womans <= limit && mans == 0)**. Imediatamente ao sair do laço de espera, o contador de espera para o sexo da pessoa é decrementado **(womansWaiting--, mansWaiting--)**. 

Em seguida, o contador de uso do banheiro para o sexo da pessoa é incrementado, em conjunto com o contador de pessoas consecutivas **(womans++, mans++, consecutive++)**. Por fim, é feito o teste de troca de prioridade quando o número de pessoas consecutivas alcança o limite permitido **(consecutive >= maxConsecutive)**.

A lógica de espera dos homens é muito semelhante das mulheres, porém as variáveis de controle são invertidas para refletir as regras para este gênero.

```java
    public synchronized void enterBathroom(Person person) {
        int limit = maxConsecutive - 1;
        try {
            switch (person.getSex()) {
                case FEMALE:
                    womansWaiting++;
                    while (priority == Sex.MALE || womans > limit || mans != 0) {
                        if (mansWaiting == 0 && womans <= limit && mans == 0) {
                            changePriority(Sex.FEMALE);
                            break;
                        }
                        this.wait();
                    }
                    womansWaiting--;
                    
                    womans++;
                    consecutive++;
                    
                    if (consecutive >= maxConsecutive)
                        changePriority(Sex.MALE);
                    break;
                case MALE:
                    mansWaiting++;
                    while (priority == Sex.FEMALE || mans > limit || womans != 0) {
                        if (womansWaiting == 0 && mans <= limit && womans == 0) {
                            changePriority(Sex.MALE);
                            break;
                        }
                        this.wait();
                    }
                    mansWaiting--;
                    
                    mans++;
                    consecutive++;
                    
                    if (consecutive == maxConsecutive)
                        changePriority(Sex.FEMALE);
                    break;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

Por fim, o método de sair do banheiro é bem simple. O contador de pessoas usando o banheiro para o sexo da pessoa é decrementado **(womans--, mans--)**, e o último da fila faz uma notificação de banheiro vazio.

```java
    public synchronized void exitBathroom(Person person) {
        switch (person.getSex()) {
            case FEMALE:
                womans--;
                break;
            case MALE:
                mans--;
                break;
        }
        if (mans == 0 || womans == 0)
            this.notifyAll();
    }
```

## Referências
- [Versão em GO](https://blog.ksub.org/bytes/2016/04/17/the-unisex-bathroom-problem/)
- [Video com um pseudocódigo com semaforos](https://www.youtube.com/watch?v=FMCKFOyud-c)
