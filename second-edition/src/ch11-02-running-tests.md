## Controlando Como os Testes São Executados

Assim como `cargo run` compila seu código e, em seguida, executa o binário 
resultante,`cargo test` compila seu código no modo de teste e executa o binário 
de teste resultante. Você pode especificar opções de linha de comando para alterar 
o comportamento padrão do `cargo test`. Por exemplo, o comportamento padrão do 
binário produzido pelo `cargo test` é executar todos os testes em paralelo e 
capturar a saída gerada durante as execuções de teste, impedindo que a saída seja 
exibida e facilitando a leitura da saída relacionada aos resultados do teste .

Algumas opções de linha de comando vão para `cargo test` e outras vão para o teste 
binário resultante. Para separar esses dois tipos de argumentos, você lista os 
argumentos que vão para `cargo test`, seguidos pelo separador `--` e depois os que 
vão para o binário de teste. Executar `cargo test --help` exibe as opções que você 
pode usar com `cargo test` e executar `cargo test -- --help` exibe as opções que 
você pode usar após o separador `--`.

### Executando Testes em Paralelo ou Consecutivamente

Quando você executa vários testes, por padrão, eles são executados em paralelo 
usando threads. Isso significa que os testes serão concluídos mais rapidamente, 
para que você possa obter um feedback mais rápido sobre se o seu código está 
funcionando ou não. Como os testes estão sendo executados ao mesmo tempo, verifique 
se eles não dependem um do outro ou de qualquer estado compartilhado, incluindo um 
ambiente compartilhado, como o diretório de trabalho atual ou as variáveis de ambiente.

Por exemplo, digamos que cada um de seus testes execute algum código que cria 
um arquivo no disco chamado *test-output.txt* e grave alguns dados nesse arquivo. 
Em seguida, cada teste lê os dados nesse arquivo e afirma que o arquivo contém um 
valor específico, que é diferente em cada teste. Como os testes são executados ao 
mesmo tempo, um teste pode substituir o arquivo entre quando outro teste grava e 
lê o arquivo. O segundo teste falhará, não porque o código esteja incorreto, mas 
porque os testes interferiram um com o outro durante a execução em paralelo. 
Uma solução é garantir que cada teste grave em um arquivo diferente; outra 
solução é executar os testes um de cada vez.

Se você não deseja executar os testes em paralelo ou se deseja um controle mais 
refinado sobre o número de threads usadas, pode enviar o sinalizador (flag) 
`--test-threads` e o número de threads que deseja usar para o teste binário. 
Veja o seguinte exemplo:

```text
$ cargo test -- --test-threads=1
```

Definimos o número de threads de teste como `1`, dizendo ao programa para não 
usar nenhum paralelismo. A execução dos testes usando um encadeamento levará 
mais tempo do que em paralelo, mas os testes não interferirão entre si se eles 
compartilharem o estado.

### Mostrando a Saída da Função

Por padrão, se um teste for aprovado, a biblioteca de testes de Rust captura 
qualquer coisa impressa na saída padrão. Por exemplo, se chamarmos `println!` 
em um teste, e o teste for aprovado, não veremos a saída `println!` no terminal; 
veremos apenas a linha que indica que o teste foi aprovado. Se um teste falhar, 
veremos o que foi impresso na saída padrão com o restante da mensagem de falha.

Como exemplo, a Lista 11-10 tem uma função boba que imprime o valor de seu parâmetro 
e retorna 10, além de um teste que passa e um teste que falha.

<span class="filename">Nome do arquivo: src/lib.rs</span>

```rust
fn prints_and_returns_10(a: i32) -> i32 {
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```

<span class="caption">Listagen 11-10: Testes para uma função que chama
`println!`</span>

Quando executamos esses testes com `cargo test`, veremos a seguinte saída:

```text
running 2 tests
test tests::this_test_will_pass ... ok
test tests::this_test_will_fail ... FAILED

failures:

---- tests::this_test_will_fail stdout ----
        I got the value 8
thread 'tests::this_test_will_fail' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:8
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out
```

Observe que em nenhum lugar desta saída vemos `I got the value 4`, que é o que é 
impresso quando o teste que passa é executado. Essa saída foi capturada. A saída 
do teste que falhou, `I got the value 8`, aparece na seção da saída do resumo do 
teste, que também mostra a causa da falha do teste.

Se também queremos ver os valores impressos para passar nos testes, podemos 
desativar o comportamento de captura de saída usando o sinalizador `--nocapture`:

```text
$ cargo test -- --nocapture
```

Quando executamos os testes na Lista 11-10 novamente com o sinalizador `--nocapture`, 
vemos a seguinte saída:

```text
running 2 tests
I got the value 4
I got the value 8
test tests::this_test_will_pass ... ok
thread 'tests::this_test_will_fail' panicked at 'assertion failed: `(left == right)`
  left: `5`,
 right: `10`', src/lib.rs:19:8
note: Run with `RUST_BACKTRACE=1` for a backtrace.
test tests::this_test_will_fail ... FAILED

failures:

failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out
```

Observe que a saída para os testes e os resultados dos testes são intercalados; 
o motivo é que os testes estão sendo executados em paralelo, como falamos na seção 
anterior. Tente usar a opção `--test-threads=1` e o sinalizador `--nocapture` e 
veja como é a saída então!

### Executando um Subconjunto de Testes por Nome

Às vezes, a execução de um conjunto de testes completo pode levar muito tempo. Se 
você estiver trabalhando no código em uma área específica, convém executar apenas 
os testes pertencentes a esse código. Você pode escolher quais testes executar ao 
passar no `cargo test` o nome ou nomes dos testes que deseja executar como argumento.

Para demonstrar como executar um subconjunto de testes, criaremos três testes para nossa 
função `add_two`, conforme mostrado na Listagem 11-11, e escolheremos quais executar:

<span class="filename">Nome do arquivo: src/lib.rs</span>

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn add_three_and_two() {
        assert_eq!(5, add_two(3));
    }

    #[test]
    fn one_hundred() {
        assert_eq!(102, add_two(100));
    }
}
```

<span class="caption">Listagem 11-11: Três testes com três nomes diferentes</span>

Se executarmos os testes sem passar nenhum argumento, como vimos anteriormente, 
todos os testes serão executados em paralelo:

```text
running 3 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok
test tests::one_hundred ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

#### Executando Testes Únicos

Podemos passar o nome de qualquer função de teste para `cargo test` para executar 
apenas esse teste:

```text
$ cargo test one_hundred
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-06a75b4a1f2515e9

running 1 test
test tests::one_hundred ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out
```

Somente o teste com o nome `one_hundred` foi executado; os outros dois testes 
não correspondiam a esse nome. A saída do teste nos permite saber que tivemos 
mais testes do que o que esse comando executou exibindo `2 filtered out` 
(2 filtrado) no final da linha de resumo.

Não podemos especificar os nomes de vários testes dessa maneira; somente o primeiro 
valor dado ao `cargo test` será usado. Mas há uma maneira de executar vários testes.

#### Filtragem para Executar Vários Testes

Podemos especificar parte de um nome de teste e qualquer teste cujo nome corresponda 
a esse valor será executado. Por exemplo, como dois nomes de nossos testes contêm `add`, 
podemos executá-los executando `cargo test add`:

```text
$ cargo test add
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-06a75b4a1f2515e9

running 2 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out
```

Este comando executou todos os testes com `add` no nome e filtrou o teste chamado 
`one_hundred`. Observe também que o módulo no qual os testes aparecem se torna parte 
do nome do teste, para que possamos executar todos os testes em um módulo filtrando 
o nome do módulo.

### Ignorando Alguns Testes, a Menos que Especificamente Solicitado

Às vezes, alguns testes específicos podem levar muito tempo para serem executados; 
portanto, você pode excluí-los durante a maioria das execuções do `cargo test`. Em vez 
de listar como argumentos todos os testes que você deseja executar, você pode anotar 
os testes demorados usando o atributo `ignore` para excluí-los, como mostrado aqui:

<span class="filename">Nome do arquivo: src/lib.rs</span>

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

Após `#[test]`, adicionamos a linha `#[ignore]` ao teste que queremos excluir. 
Agora, quando executamos nossos testes, o `it_works` é executado, mas o 
`expensive_test` não:

```text
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.24 secs
     Running target/debug/deps/adder-ce99bcc2479f4607

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out
```

A função `expensive_test` é listada como `ignored`. Se queremos executar apenas 
os testes ignorados, podemos usar `cargo test - --ignored`:

```text
$ cargo test -- --ignored
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-ce99bcc2479f4607

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out
```

Ao controlar quais testes são executados, você pode garantir que os resultados do 
`cargo test` sejam rápidos. Quando você está em um ponto em que faz sentido verificar 
os resultados dos testes `ignored` e tem tempo para aguardar os resultados, você pode 
executar o `cargo test -- --ignored`.