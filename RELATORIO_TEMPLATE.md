# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: 9 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

```
A diferença entre os métodos de printf() e write() está na forma em com que as syscalls (chamadas de sistema) são geradas durante a execução do código. 
Diferentemente do write(), o printf() não realiza obrigatoriamente uma syscall a cada chamada, pois ele possui um sistema de buffer que o permite armazenar instruções de impressão de textos, até um certo limite, até que o syscall seja realizado.
```

**3. Qual método é mais previsível? Por quê você acha isso?**

```
O buffer do printf() o torna uma alternativa mais rápida do que o write(), por não exigir a troca de contexto (pro kernel) a cada chamada, porém o torna menos previsível já que uma execução do método nem sempre equivale a uma syscall, como por exemplo se a linha completar com /n ele pode realizar uma sycall instantaneamente então depende da situação justamente por não ser uma relação 1:1.
Vale ressaltar que no fundo o printf() também utiliza o write() só que há um processo de buffer que otimiza a perfomance do código.
```

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: 3
- Bytes lidos: 127

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

```
O file descriptor utilizado para realizar a leitura do arquivo txt é o 3, ele não começou em 0, 1 ou 2 pois estes identificadores já são reservados pelo sistema operacional para realizar operações criticas

0 - stdin: entrada padrão (normalmente o teclado) 
1 - stdout: saída padrão (normalmente o terminal) 
2 - stderr: saída de erro
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
Se o retorno da syscall read() for igual a 0, indica que o arquivo foi lido completamente.
```

**3. Por que verificar retorno de cada syscall?**

```
Porque é pelo retorno do syscall que é possível analisar se a operação requerida ao kernel deu certo (retorno de sucesso) ou não (erro), e em caso de erros é o tipo de retorno que nos permite realizar o tratamenot correto de erros.
```

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: 25 (esperado: 25)
- Caracteres: 1300
- Chamadas read(): 22
- Tempo: 0.000478 segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |        88       |  0.000201 |
| 64          |        22       |  0.000085 |
| 256         |        7        |  0.000068 |
| 1024        |        3        |  0.000067 |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
Quanto menor o buffer, maior o número de syscalls necessários para ler o arquivo completo, pois o buffer armazena o conteúdo para cada chamada do read() e quanto menor o espaço armazenado maior será o numero de syscalls necessárias para ler o conteúdo 
```

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

```
Nem todas, quando a leitura do arquivo está no final o Buffer_size pode retornar uma quantidade de bytes menor do que as outras chamadas anteriores (que utilizavam todo o espaço do buffer_size uma vez que o arquivo não foi lido completamente)
```

**3. Qual é a relação entre syscalls e performance?**

```
Quanto maior a qtde de syscalls realizada, pior a perfomance do processo, pois quando um syscall é realizado o kernel precisa assumir o controle do processo para realizar a operação desejada, isto é, ele interrompe o processo atual armazena os dados atuais do processo para poder retormar depois e quando o so finaliza ele volta para o processo inicial, toda essa troca de controle é chamada de troca de contexto o que leva tempo e gera queda de performance
```

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: 1364
- Operações: 7
- Tempo: 0.000256 segundos
- Throughput: 5203.25 KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [X] Idênticos [ ] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
A comparação de bytes_escritos e bytes_lidos indica se a cópia do conteúdo do arquivo de origem foi realizada de forma integral para o arquivo destino, se for igual, a cópia do arquivo foi realizada com sucesso. Se a quantidade de bytes escritos for menor que os bytes lidos, a cópia foi construida parcialmente, isto é, faltam conteúdos do arquivo origem serem transferidos para o arquivo destino.
```

**2. Que flags são essenciais no open() do destino?**

```
São elas: 
O_WRONLY: dá permissão para o file pointer para somente escrever no arquivo
O_CREAT: se o arquivo deste nome não existir no momento do open(), um arquivo será criado 
O_TRUNC: caso o arquivo já exista ele apaga o conteúdo antigo do arquivo no momento do open, impedindo que possíveis duplicidades ocorram

```

**3. O número de reads e writes é igual? Por quê?**

```
Não são iguais, o número de read() é de 8 enquanto o número de write() é de 18. Isso acontece porque o syscall write não garante que todos os dados do buffer serão escritos no arquivo destino em apenas uma só chamada, eventualmente ele poderá dividir o conteúdo a ser gravado em várias partes menores a depender do contexto. Portanto neste caso o totalizador de write é maior do que o totalizar do read.
Além de que, mesmo se o read() e write() fossem perfeitamente alinhados, ainda poderia existir uma pequena diferença dado a maneira como o código está escrito, já que para o while quebrar seu looping, seria necessário o read() rodar mais uma vez para verificar se o valor retornado é 0 (indicando fim do arquivo), neste caso eu teria uma chamada adicional do read() em relação ao write()
```

**4. Como você saberia se o disco ficou cheio?**

```
Se acontecer do disco ficar cheio durante a execução, a syscall write() terá um retorno de erro (ou seja, -1) indicando que a gravação de conteúdo no arquivo não poderá ser completada dado a falta de espaço no disco. Esse erro em específico também aparecerá como "Erro na escrita: No space left on device" ao chamar a função perror() que descreve o erro encontrado
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
[Sua análise aqui]
```

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
[Sua análise aqui]
```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
[Sua análise aqui]
```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
[Sua análise aqui]
```

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** _____

**Por que você acha que foi mais rápido?**

```
[Sua análise aqui]
```

---

## 📤 Entrega
Certifique-se de ter:
- [ ] Todos os códigos com TODOs completados
- [ ] Traces salvos em `traces/`
- [ ] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
