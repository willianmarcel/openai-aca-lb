![Balanceamento inteligente de carga](./images/intro-loadbalance.png)

# :rocket: Balanceador inteligente de carga para endpoints OpenAI

Muitos provedores de serviços, incluindo OpenAI, geralmente definem limites no número de chamadas que podem ser feitas. No caso do Azure OpenAI, existem limites de tokens (TPM ou tokens por minuto) e limites no número de requisições por minuto (RPM). Quando um servidor começa a ficar sem recursos ou os limites de serviço são esgotados, o provedor pode emitir um código de status HTTP 429 ou TooManyRequests, e também um cabeçalho de resposta Retry-After indicando quanto tempo você deve esperar até tentar a próxima requisição.

A solução apresentada aqui é parte de uma abordagem abrangente que leva em consideração aspectos como um bom design de UX/workflow, adição de lógica de resiliência e tratamento de falhas da aplicação, consideração dos limites de serviço, escolha do modelo certo para o trabalho, políticas de API, configuração de logging e monitoramento, entre outras considerações. Esta solução expõe de forma transparente um único endpoint para suas aplicações, mantendo uma lógica eficiente para consumir dois ou mais backends OpenAI ou qualquer API baseada em disponibilidade e prioridade.

Ela é construída usando o framework de proxy reverso de alta performance [YARP C#](https://github.com/microsoft/reverse-proxy) da Microsoft. No entanto, você não precisa entender C# para usá-la, você pode simplesmente construir a imagem Docker fornecida.
Esta é uma solução alternativa ao [Balanceador inteligente de carga OpenAI do API Management](https://github.com/Azure-Samples/openai-apim-lb/), com a mesma lógica.

## :sparkles: Por que você chama isso de "inteligente" e diferente de balanceadores de carga round-robin?

Um dos componentes-chave do tratamento de throttling do OpenAI é estar ciente do código de status HTTP 429 (Too Many Requests). Existem limitações de [Tokens-Por-Minuto e Requisições-Por-Minuto](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota?tabs=rest#understanding-rate-limits) no Azure OpenAI. Ambas as situações retornarão o mesmo código de erro 429.

Junto com esse código de status HTTP 429, o Azure OpenAI também retornará um cabeçalho de resposta HTTP chamado "Retry-After", que é o número de segundos que essa instância ficará indisponível antes de começar a aceitar requisições novamente.

Esses erros são normalmente tratados no lado do cliente pelos SDKs. Isso funciona muito bem se você tem um único endpoint de API. No entanto, para múltiplos endpoints OpenAI (usados para fallback) você precisaria gerenciar a lista de URLs no lado do cliente também, o que não é ideal.

O que torna esta solução diferente das outras é que ela está ciente do "Retry-After" e erros 429 e inteligentemente envia o tráfego para outros backends OpenAI que não estão atualmente com throttling. Você pode até ter uma ordem de prioridade em seus backends, então os de maior prioridade são os que são consumidos primeiro enquanto não estão com throttling. Quando o throttling ocorre, ele fará fallback para backends de menor prioridade enquanto os de maior prioridade estão esperando para se recuperar.

Outra funcionalidade importante: não há intervalo de tempo entre tentativas para chamar diferentes backends. Muitos outros balanceadores de carga OpenAI por aí configuram um intervalo de espera interno (frequentemente exponencial). Embora isso seja uma boa ideia fazer no lado do cliente, fazer um balanceador de carga do lado do servidor esperar não é uma boa prática porque você segura seu cliente e consome mais capacidade de servidor e rede durante esse tempo de espera. Tentativas no lado do servidor devem ser imediatas e para um endpoint diferente.

Confira este diagrama para um entendimento mais fácil:

![normal!](/images/apim-loadbalancing-active.png "Cenário normal")

![throttling!](/images/apim-loadbalancing-throttling.png "Cenário de throttling")

## :1234: Prioridades

Uma coisa que se destaca nas imagens acima é o conceito de "grupos de prioridade". Por que temos isso? Isso porque você pode querer consumir toda sua quota disponível em instâncias específicas antes de fazer fallback para outras. Por exemplo, neste cenário:
- Você tem um deployment [PTU (Provisioned Throughput)](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/provisioned-throughput). Você quer consumir toda sua capacidade primeiro porque está pagando por isso, use ou não. Você pode definir essa(s) instância(s) como **Prioridade 1**
- Então você tem deployments extras usando o tier padrão S0 (modelo de consumo baseado em token) espalhados em diferentes regiões do Azure que você gostaria de usar como fallback caso sua instância PTU esteja totalmente ocupada e retornando erros 429. Aqui você não tem um modelo de preço fixo como no PTU, mas consumirá esses endpoints apenas durante o período que PTU não está disponível. Você pode defini-los como **Prioridade 2**

Outro cenário:
- Você não tem nenhum deployment PTU (provisionado) mas gostaria de ter muitos S0 (modelo de consumo baseado em token) espalhados em diferentes regiões do Azure caso você atinja throttling. Vamos assumir que suas aplicações estão principalmente nos EUA.
- Você então deploya uma instância de OpenAI em cada região dos EUA que tem capacidade OpenAI. Você pode definir essa(s) instância(s) como **Prioridade 1**
- No entanto, se todas as instâncias dos EUA estão sendo throttled, você tem outro conjunto de endpoints no Canadá, que é a região mais próxima fora dos EUA. Você pode definir essa(s) instância(s) como **Prioridade 2**
- Mesmo se o Canadá também sofrer throttling ao mesmo tempo que suas instâncias dos EUA, você pode fazer fallback para regiões europeias agora. Você pode definir essa(s) instância(s) como **Prioridade 3**
- No último caso, se todos os outros endpoints anteriores ainda estão com throttling durante o mesmo tempo, você pode até considerar ter endpoints OpenAI na Ásia como "último recurso". A latência será um pouco maior, mas ainda aceitável. Você pode definir essa(s) instância(s) como **Prioridade 4**

E o que acontece se eu tiver múltiplos backends com a mesma prioridade? Vamos assumir que tenho 3 backends OpenAI nos EUA, todos com Prioridade = 1 e todos eles não estão com throttling? Neste caso, o algoritmo escolherá aleatoriamente entre essas 3 URLs.

## :gear: Começando

O código-fonte fornece um Dockerfile, o que significa que você está livre para construir e fazer deploy para seu próprio serviço, contanto que suporte imagens de container.

### [Opção 1 - Recomendada] Deploy usando Azure Developer CLI
Fazer deploy desta solução usando o [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd) é super simples. Tudo que você precisa fazer é clonar este repositório e executar o seguinte comando, localmente (dado que você [instalou](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd) o Azure Developer CLI) e [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli):

```
azd up
```

> [!NOTE]  
> Se você estiver usando GitHub Codespaces, você precisa fazer login no AZD e AZ CLI antes de executar *azd up*:
> ```
> az login --use-device-code
> azd auth login --use-device-code
> azd up
> ```

Seu deployment criará um Azure Container Apps com três backends GPT 3.5 Turbo para fazer balanceamento de carga. Se você quiser adicionar mais, você pode simplesmente editar as variáveis de ambiente do seu Container Apps.
Cada uma das instâncias OpenAI será deployada com 30K TPM (tokens por minuto) de capacidade por padrão. Se você estiver recebendo erros de capacidade de deployment, você pode querer diminuir esse valor. Ou, se você está planejando fazer deploy de maior capacidade, você pode definir a seguinte variável AZD antes de fazer deploy:

```
azd env set OPENAI_CAPACITY 50
```

Este comando fará deploy dos três backends com 50K TPM cada. Esteja ciente das [quotas e limites regionais](https://learn.microsoft.com/en-us/azure/ai-services/openai/quotas-limits)

Quando terminar de testar, você pode derrubar todos os recursos executando

```
azd down
```

### [Opção 2] Deploy do serviço diretamente para um Azure Container Apps

Se você não se sente confortável trabalhando com imagens de container ou clonando este repositório e gostaria de uma maneira muito fácil de testar este balanceador de carga no Azure, você pode fazer deploy rapidamente para [Azure Container Apps](https://azure.microsoft.com/products/container-apps):

[![Deploy para Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fopenai-aca-lb%2Fmain%2Fazuredeploy.json)

- Clicando no botão Deploy acima, você será levado para uma página do Azure com os parâmetros necessários. Você precisa preencher os parâmetros começando com "Backend_X_" (veja abaixo em [Configurando os endpoints OpenAI](#configurando-os-endpoints-openai) para mais informações sobre o que eles fazem)
- Após o deployment terminar, vá para seu serviço Container Apps recém-criado e do menu Overview, obtenha a Application Url da sua aplicação. O formato será "https://app-[alguma-coisa].[região].azurecontainerapps.io". Esta é a URL que você chamará de suas aplicações cliente

### [Opção 3] Construir e fazer deploy como uma imagem Docker

Se você quiser clonar este repositório e construir sua própria imagem ao invés de usar a imagem pública pré-construída:

```
cd src
docker build -t aoai-smart-loadbalancing:v1 .
```

Isso usará o Dockerfile que construirá o código-fonte dentro do próprio container (não há necessidade de ter ferramentas de build .NET na sua máquina host) e então copiará a saída do build para uma nova imagem de runtime para ASP.NET 8. Apenas certifique-se de que sua versão do Docker suporta builds [multi-stage](https://docs.docker.com/build/building/multi-stage/). A imagem final terá cerca de 87 MB.

### [Opção 4] Deploy da imagem pré-construída do Docker hub

Se você não quiser construir o container a partir do código-fonte, você pode baixá-lo do registro público do Docker:

```
docker pull andredewes/aoai-smart-loadbalancing:v1
```

### Configurando os endpoints OpenAI

Após fazer deploy do seu serviço de container usando um dos métodos acima, é hora de ajustar a configuração dos seus backends OpenAI usando variáveis de ambiente.
Este é o formato esperado que você deve fornecer:

| Nome da variável de ambiente | Obrigatório | Descrição                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Exemplo                               |
|---------------------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| BACKEND_X_URL             | Sim       | A URL completa do Azure OpenAI. Substitua "_X_" pelo número do seu backend. Por exemplo, "BACKEND_1_URL" ou "BACKEND_2_URL"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | https://andre-openai.openai.azure.com |
| BACKEND_X_PRIORITY        | Sim       | A prioridade do seu endpoint OpenAI. Números menores significam maior prioridade. Substitua "_X_" pelo número do seu backend. Por exemplo, "BACKEND_1_PRIORITY" ou "BACKEND_2_PRIORITY"                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | 1                                     |
| BACKEND_X_APIKEY          | Não       | A chave API do seu endpoint OpenAI. Substitua "_X_" pelo número do seu backend. Por exemplo, "BACKEND_1_APIKEY" ou "BACKEND_2_APIKEY". Se esta configuração não for definida, tentará usar suas credenciais de ambiente, incluindo Managed Identities caso você esteja executando cargas de trabalho do Azure. Para mais detalhes, confira [DefaultAzureCredential](https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme?view=azure-dotnet#defaultazurecredential)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | 761c427c520d40c991299c66d10c6e40      |
| BACKEND_X_DEPLOYMENT_NAME | Não        | Se esta configuração for definida, a URL do cliente que chega terá o nome do deployment sobrescrito. Por exemplo, uma requisição HTTP que chega é:<br><br>https://andre-openai-eastus.openai.azure.com/openai/deployments/gpt35turbo/chat/completions?api-version=2023-07-01-preview <br><br>O **gpt35turbo** será substituído por esta configuração quando a requisição for para o seu backend. Dessa forma, você não precisa ter todos os nomes de deployment iguais em seus endpoints. Você pode até misturar diferentes modelos GPT (como GPT35 e GPT4), embora isso não seja recomendado. <br><br>Se nada for definido, a URL que chega de suas aplicações cliente será mantida igual quando enviada para o backend, não será sobrescrita. Isso significa que seus nomes de deployment devem ser iguais em seus endpoints OpenAI. | your-openai-deployment-name           |

Por exemplo, digamos que você gostaria de configurar o balanceador de carga para ter um endpoint principal (chamamos aqui de BACKEND_1). Definimos com a maior prioridade 1. E então temos dois endpoints mais como fallback caso o principal esteja com throttling... definimos então BACKEND_2 e BACKEND_3 com a mesma prioridade 2:

| Nome da variável de ambiente | Valor                                   |
|---------------------------|-----------------------------------------|
| BACKEND_1_URL             | https://andre-openai.openai.azure.com   |
| BACKEND_1_PRIORITY        | 1                                       |
| BACKEND_1_APIKEY          | 33b9996ce4644bc0893c7988bae349af        |
| BACKEND_2_URL             | https://andre-openai-2.openai.azure.com |
| BACKEND_2_PRIORITY        | 2                                       |
| BACKEND_2_APIKEY          | 412ceac74dde451e9ac12581ca50b5c5        |
| BACKEND_3_URL             | https://andre-openai-3.openai.azure.com |
| BACKEND_3_PRIORITY        | 2                                       |
| BACKEND_3_APIKEY          | 326ec768694d4d469eda2fe2c582ef8b        |

### Configurações do balanceador de carga

Esta é a lista de variáveis de ambiente que são usadas para configurar o balanceador de carga em escopo global, não apenas para endpoints backend específicos:

| Nome da variável de ambiente | Obrigatório | Descrição                                                                                                                                                                                                                                         | Exemplo |
|---------------------------|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------|
| HTTP_TIMEOUT_SECONDS      | Não        | Se definido, mudará o timeout padrão de 100 segundos ao aguardar respostas do OpenAI para algo diferente.<br>Se um timeout for atingido, o endpoint será marcado como não saudável por 10 segundos e a requisição fará fallback para outro backend. | 120     |

### Testando a solução

Para testar se tudo funciona executando algum código de sua escolha, por exemplo, este código com OpenAI Python SDK:

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://<your_load_balancer_url>",  #se você fez deploy para Azure Container Apps, será 'https://app-[alguma-coisa].[região].azurecontainerapps.io'
    api_key="nao-importa", #A api-key enviada pelos SDKs cliente será sobrescrita pelas configuradas nas variáveis de ambiente backend
    api_version="2023-12-01-preview"
)

response = client.chat.completions.create(
    model="<your_openai_deployment_name>",
    messages=[
        {"role": "system", "content": "Você é um assistente útil."},
        {"role": "user", "content": "Qual é a primeira letra do alfabeto?"}
    ]
)
print(response)
```

### Escalabilidade vs Confiabilidade
Esta solução aborda tanto preocupações de escalabilidade quanto confiabilidade ao permitir que suas quotas totais do Azure OpenAI aumentem e fornecer failovers do lado do servidor de forma transparente para suas aplicações. No entanto, se você está procurando puramente uma maneira de aumentar quotas padrão, ainda recomendaria que você seguisse a orientação oficial para [solicitar um aumento de quota](https://learn.microsoft.com/azure/ai-services/openai/quotas-limits#how-to-request-increases-to-the-default-quotas-and-limits).

### Múltiplas instâncias do balanceador de carga
Esta solução usa a memória local para armazenar o estado de saúde dos endpoints. Isso significa que cada instância terá sua própria visão do estado de throttling de cada endpoint OpenAI. O que pode acontecer durante o runtime é isso:
- Instância 1 recebe uma requisição do cliente e obtém um erro 429 do backend 1. Ela marca esse backend como indisponível por X segundos e então reencaminha essa requisição do cliente para o próximo backend
- Instância 2 recebe uma requisição do cliente e envia essa requisição novamente para o backend 1 (já que sua lista local em cache de backends não tinha a informação da instância 1 quando ela marcou como throttled). Backend 1 responderá com erro 429 novamente e a instância 2 também marcará como indisponível e reencaminhará a requisição para o próximo backend

Então, pode ocorrer que internamente, diferentes instâncias tentem rotear para backends com throttling e precisem tentar novamente para outro backend. Eventualmente, todas as instâncias estarão em sincronia novamente a um pequeno custo de roundtrips desnecessários para endpoints com throttling.
Honestamente acho que esse é um preço muito pequeno a pagar, mas se você quiser resolver isso, sempre pode mudar o código-fonte para usar um cache externo compartilhado como Redis, então todas as instâncias compartilharão o mesmo objeto em cache.

Tendo isso em mente, seja cuidadoso quando configurar seu serviço de container host quando se trata de escalabilidade. Por exemplo, as regras de escala padrão em um Azure Container Apps são o número de requisições HTTP simultâneas: se for maior que 10, criará outra instância de container. Este efeito é indesejável para o balanceador de carga pois criará muitas instâncias, e é por isso que o botão Quick Deploy neste repositório muda esse comportamento padrão para escalar o container apenas quando o uso de CPU for maior que 50%.

### Logging
As funcionalidades de logging padrão vindas do [YARP](https://microsoft.github.io/reverse-proxy/articles/diagnosing-yarp-issues.html) não são alteradas aqui, ainda são aplicáveis. Por exemplo, você deve ver essas linhas de log sendo impressas no console do container (Stdout) quando requisições são redirecionadas com sucesso para os backends:

```
info: Yarp.ReverseProxy.Forwarder.HttpForwarder[9]
info: Proxying to https://andre-openai-eastus.openai.azure.com/openai/deployments/gpt35turbo/chat/completions?api-version=2023-07-01-preview HTTP/2 RequestVersionOrLower 
info: Yarp.ReverseProxy.Forwarder.HttpForwarder[56]
info: Received HTTP/2.0 response 200.
info: Yarp.ReverseProxy.Forwarder.HttpForwarder[9]
info: Proxying to https://andre-openai-eastus.openai.azure.com/openai/deployments/gpt35turbo/chat/completions?api-version=2023-07-01-preview HTTP/2 RequestVersionOrLower 
info: Yarp.ReverseProxy.Forwarder.HttpForwarder[56]
info: Received HTTP/2.0 response 200.
```

Agora, estes são exemplos de logs gerados quando o balanceador de carga recebe um erro 429 do backend OpenAI:

```
info: Yarp.ReverseProxy.Forwarder.HttpForwarder[56]
info: Received HTTP/2.0 response 429.
warn: Yarp.ReverseProxy.Health.DestinationHealthUpdater[19]
warn: Destination `BACKEND_4` marked as 'Unhealthy` by the passive health check is scheduled for a reactivation in `00:00:07`.
```

Observe que ele lê o valor vindo no cabeçalho "Retry-After" da resposta OpenAI e marca esse backend como Não Saudável e também imprime quanto tempo levará para ser reativado. Neste caso, 7 segundos.

E esta é a linha de log que aparece depois que esse tempo transcorre:

```
info: Yarp.ReverseProxy.Health.DestinationHealthUpdater[20]
info: Passive health state of the destination `BACKEND_4` is reset to 'Unknown`.
```

É OK que diga que o status foi resetado para "Unknown". Isso significa que esse backend estará ativamente recebendo requisições HTTP e seu estado interno será atualizado para Saudável se receber uma resposta 200 do backend OpenAI na próxima vez. Isso é chamado de Verificação passiva de saúde e é uma [funcionalidade do YARP](https://microsoft.github.io/reverse-proxy/articles/dests-health-checks.html#passive-health-checks).

## :question: FAQ

### O que acontece se todos os backends estiverem com throttling ao mesmo tempo?
Nesse caso, o balanceador de carga roteará para um backend aleatório na lista. Como esse endpoint está com throttling, retornará o mesmo erro 429 que o backend OpenAI. É por isso que **ainda é importante para sua aplicação cliente/SDKs ter uma lógica para lidar com tentativas**, mesmo que devam ser muito menos frequentes.

### Ler a lógica C# é difícil para mim. Você pode descrevê-la em português simples?
Claro. É assim que funciona quando o balanceador de carga recebe uma nova requisição:

1. Da lista de backends definida nas variáveis de ambiente, ele escolherá um backend usando esta lógica:
   1. Seleciona a maior prioridade (menor número) que não está atualmente com throttling. Se encontrar mais de um backend saudável com a mesma prioridade, selecionará aleatoriamente um deles
2. Envia a requisição para a URL do backend escolhido
    1. Se o backend responder com sucesso (código de status HTTP 200), a resposta é passada para o cliente e o fluxo termina aqui
    2. Se o backend responder com erro 429 ou 5xx
        1. Lerá o cabeçalho de resposta HTTP "Retry-After" para ver quando ficará disponível novamente
        2. Então, marca essa URL de backend específica como com throttling e também salva que horas ficará saudável novamente
        3. Se ainda houver outros backends disponíveis no pool, executa novamente a lógica para selecionar outro backend (volta ao ponto 1. novamente e assim por diante)
        4. Se não houver backends disponíveis (todos estão com throttling), enviará a requisição do cliente para o primeiro backend definido na lista e retornará sua resposta

## :link: Artigos relacionados
- A mesma lógica de balanceador de carga mas usando Azure API Management: [Balanceamento Inteligente de Carga para Azure OpenAI com Azure API Management](https://github.com/Azure-Samples/openai-apim-lb/)
