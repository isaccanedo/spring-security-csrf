## Um guia para proteção CSRF em Spring Security

# 1. Visão Geral
Neste breve tutorial, discutiremos ataques CSRF de falsificação de solicitação entre sites e como evitá-los usando Spring Security.

# 2. Dois ataques simples de CSRF
Existem várias formas de ataques CSRF - vamos discutir algumas das mais comuns.

### 2.1. Exemplos GET
Vamos considerar a seguinte solicitação GET usada por um usuário conectado para transferir dinheiro para uma conta bancária específica “1234”:

```
GET http://bank.com/transfer?accountNo=1234&amount=100
```

Se o invasor quiser transferir dinheiro da conta de uma vítima para sua própria conta - “5678” - ele precisa fazer a vítima acionar a solicitação:

```
GET http://bank.com/transfer?accountNo=5678&amount=1000
```

Existem várias maneiras de fazer isso acontecer:

```Link:``` O invasor pode convencer a vítima a clicar neste link, por exemplo, para executar a transferência:

```
<a href="http://bank.com/transfer?accountNo=5678&amount=1000">
Show Kittens Pictures
</a>
```

```Imagem:``` o invasor pode usar uma tag ```<img />``` com o URL de destino como fonte da imagem - então o clique nem é necessário. A solicitação será executada automaticamente quando a página for carregada:

```
<img src="http://bank.com/transfer?accountNo=5678&amount=1000"/>
```

### 2.2. Exemplo POST
Se a solicitação principal precisa ser uma solicitação POST - por exemplo:

```
POST http://bank.com/transfer
accountNo=1234&amount=100
```

Em seguida, o invasor precisa que a vítima execute um procedimento semelhante:

```
POST http://bank.com/transfer
accountNo=5678&amount=1000
```

Nem o <a> nem o ```<img />``` funcionarão neste caso. O invasor precisará de um ```<form>``` - da seguinte maneira:

```
<form action="http://bank.com/transfer" method="POST">
    <input type="hidden" name="accountNo" value="5678"/>
    <input type="hidden" name="amount" value="1000"/>
    <input type="submit" value="Show Kittens Pictures"/>
</form>
```

No entanto, o formulário pode ser enviado automaticamente usando Javascript - da seguinte forma:

```
<body onload="document.forms[0].submit()">
<form>
...
```

### 2.3. Simulação Prática
Agora que entendemos como é um ataque CSRF, vamos simular esses exemplos em um aplicativo Spring.

Vamos começar com uma implementação de controlador simples - o BankController:

```
@Controller
public class BankController {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @RequestMapping(value = "/transfer", method = RequestMethod.GET)
    @ResponseBody
    public String transfer(@RequestParam("accountNo") int accountNo, 
      @RequestParam("amount") final int amount) {
        logger.info("Transfer to {}", accountNo);
        ...
    }

    @RequestMapping(value = "/transfer", method = RequestMethod.POST)
    @ResponseStatus(HttpStatus.OK)
    public void transfer2(@RequestParam("accountNo") int accountNo, 
      @RequestParam("amount") final int amount) {
        logger.info("Transfer to {}", accountNo);
        ...
    }
}
```

E também vamos ter uma página HTML básica que acione a operação de transferência bancária:

```
<html>
<body>
    <h1>CSRF test on Origin</h1>
    <a href="transfer?accountNo=1234&amount=100">Transfer Money to John</a>
	
    <form action="transfer" method="POST">
        <label>Account Number</label> 
        <input name="accountNo" type="number"/>

        <label>Amount</label>         
        <input name="amount" type="number"/>

        <input type="submit">
    </form>
</body>
</html>
```

Esta é a página do aplicativo principal, rodando no domínio de origem.

Observe que simulamos um GET por meio de um link simples e um POST por meio de um ```<form>``` simples.

Agora - vamos ver como a página do invasor ficaria:

```
<html>
<body>
    <a href="http://localhost:8080/transfer?accountNo=5678&amount=1000">Show Kittens Pictures</a>
    
    <img src="http://localhost:8080/transfer?accountNo=5678&amount=1000"/>
	
    <form action="http://localhost:8080/transfer" method="POST">
        <input name="accountNo" type="hidden" value="5678"/>
        <input name="amount" type="hidden" value="1000"/>
        <input type="submit" value="Show Kittens Picture">
    </form>
</body>
</html>
```

Esta página será executada em um domínio diferente - o domínio do invasor.

Por fim, vamos executar os dois aplicativos - o aplicativo original e o do invasor - localmente e acessar a página original primeiro:

```
http://localhost:8081/spring-rest-full/csrfHome.html
```

Então, vamos acessar a página do invasor:

```
http://localhost:8081/spring-security-rest/api/csrfAttacker.html
```

Rastreando as solicitações exatas que se originam desta página do invasor, seremos capazes de detectar imediatamente a solicitação problemática, atingindo o aplicativo original e totalmente autenticado.

# 3. Configuração de segurança Spring
Para usar a proteção CSRF do Spring Security, primeiro precisamos nos certificar de que usamos os métodos HTTP adequados para qualquer coisa que modifique o estado (PATCH, POST, PUT e DELETE - não GET).

### 3.1. Configuração Java
A proteção CSRF é ativada por padrão na configuração Java. Ainda podemos desativá-lo se precisarmos:

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
      .csrf().disable();
}
```

### 3.2. Configuração XML
Na configuração XML mais antiga (antes do Spring Security 4), a proteção CSRF foi desabilitada por padrão e poderíamos habilitá-la da seguinte maneira:

```
<http>
    ...
    <csrf />
</http>
```

A partir do Spring Security 4.x - a proteção CSRF é ativada por padrão na configuração XML também; podemos, é claro, ainda desativá-lo se precisarmos:

```
<http>
    ...
    <csrf disabled="true"/>
</http>
```

### 3.3. Parâmetros de Formulários Extra
Finalmente, com a proteção CSRF ativada no lado do servidor, precisaremos incluir o token CSRF em nossas solicitações no lado do cliente também:

```
<input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>
```

### 3.4. Usando JSON
Não podemos enviar o token CSRF como parâmetro se estivermos usando JSON; em vez disso, podemos enviar o token dentro do cabeçalho.

Primeiro, precisaremos incluir o token em nossa página - e para isso, podemos usar metatags:

```
<meta name="_csrf" content="${_csrf.token}"/>
<meta name="_csrf_header" content="${_csrf.headerName}"/>
```

Em seguida, construiremos o cabeçalho:

```
var token = $("meta[name='_csrf']").attr("content");
var header = $("meta[name='_csrf_header']").attr("content");

$(document).ajaxSend(function(e, xhr, options) {
    xhr.setRequestHeader(header, token);
});
```

# 4. Teste CSRF desativado
Com tudo isso instalado, faremos alguns testes.

Vamos primeiro tentar enviar uma solicitação POST simples quando o CSRF está desativado:

```
@ContextConfiguration(classes = { SecurityWithoutCsrfConfig.class, ...})
public class CsrfDisabledIntegrationTest extends CsrfAbstractIntegrationTest {

    @Test
    public void givenNotAuth_whenAddFoo_thenUnauthorized() throws Exception {
        mvc.perform(
          post("/foos").contentType(MediaType.APPLICATION_JSON)
                       .content(createFoo())
          ).andExpect(status().isUnauthorized());
    }

    @Test 
    public void givenAuth_whenAddFoo_thenCreated() throws Exception {
        mvc.perform(
          post("/foos").contentType(MediaType.APPLICATION_JSON)
                       .content(createFoo())
                       .with(testUser())
        ).andExpect(status().isCreated()); 
    } 
}
```

Como você deve ter notado, estamos usando uma classe base para manter a lógica auxiliar de teste comum - o CsrfAbstractIntegrationTest:

```
@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
public class CsrfAbstractIntegrationTest {
    @Autowired
    private WebApplicationContext context;

    @Autowired
    private Filter springSecurityFilterChain;

    protected MockMvc mvc;

    @Before
    public void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context)
                             .addFilters(springSecurityFilterChain)
                             .build();
    }

    protected RequestPostProcessor testUser() {
        return user("user").password("userPass").roles("USER");
    }

    protected String createFoo() throws JsonProcessingException {
        return new ObjectMapper().writeValueAsString(new Foo(randomAlphabetic(6)));
    }
}
```

Observe que, quando o usuário tinha as credenciais de segurança corretas, a solicitação foi executada com sucesso - nenhuma informação extra foi necessária.

Isso significa que o invasor pode simplesmente usar qualquer um dos vetores de ataque discutidos anteriormente para comprometer facilmente o sistema.

# 5. Teste de CSRF habilitado
Agora, vamos habilitar a proteção CSRF e ver a diferença:

```
@ContextConfiguration(classes = { SecurityWithCsrfConfig.class, ...})
public class CsrfEnabledIntegrationTest extends CsrfAbstractIntegrationTest {

    @Test
    public void givenNoCsrf_whenAddFoo_thenForbidden() throws Exception {
        mvc.perform(
          post("/foos").contentType(MediaType.APPLICATION_JSON)
                       .content(createFoo())
                       .with(testUser())
          ).andExpect(status().isForbidden());
    }

    @Test
    public void givenCsrf_whenAddFoo_thenCreated() throws Exception {
        mvc.perform(
          post("/foos").contentType(MediaType.APPLICATION_JSON)
                       .content(createFoo())
                       .with(testUser()).with(csrf())
         ).andExpect(status().isCreated());
    }
}
```

Agora, como este teste está usando uma configuração de segurança diferente - uma que tem a proteção CSRF ativada.

Agora, a solicitação POST simplesmente falhará se o token CSRF não estiver incluído, o que, é claro, significa que os ataques anteriores não são mais uma opção.

Finalmente, observe o método csrf() no teste; isso cria um RequestPostProcessor que preencherá automaticamente um token CSRF válido na solicitação para fins de teste.

# 6. Conclusão
Neste artigo, discutimos alguns ataques CSRF e como evitá-los usando Spring Security.

Este é um projeto baseado em Maven, portanto, deve ser fácil de importar e executar como está.