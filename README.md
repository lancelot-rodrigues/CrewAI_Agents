# Projeto de Coleta de Dados de E-commerce com CrewAI

Este projeto implementa um sistema multiagente usando `CrewAI` e os modelos da OpenAI para automatizar a coleta de dados de cartuchos de tinta HP em grandes sites de varejo. O sistema é capaz de buscar produtos, extrair informações relevantes e consolidar os dados em um arquivo `.csv`.

Link do Colab: https://colab.research.google.com/drive/1kwhsxvu1RbMSeMq1vjB3GGyhMQfl7Xkp#scrollTo=e8KY009pDUSO

---

## 1. Decisões de Arquitetura e Engenharia

A estrutura do projeto foi definida com base na modularidade e, principalmente, na robustez da coleta.

### A. Escolha do `CrewAI` para Orquestração

Em vez de um script Python linear, optamos por uma arquitetura de agentes com `CrewAI` (utilizando os modelos da OpenAI como "cérebro"). Isso nos permitiu "dividir para conquistar":

1. **Agente Pesquisador (`product_search_agent`):** Sua única responsabilidade é interagir com a web. Ele é especializado em encontrar os produtos solicitados.  
2. **Agente Extrator (`data_extractor_agent`):** Este agente é o especialista em Processamento de Linguagem Natural (PLN). Ele não sabe *como* buscar na web, mas sabe *ler* os dados brutos (JSON/texto) fornecidos pelo pesquisador e extrair as informações-chave.  
3. **Agente Consolidador (`csv_consolidator_agent`):** Sua função é puramente administrativa: pegar os dados estruturados do extrator e formatá-los corretamente no padrão CSV.

Essa separação torna o sistema mais fácil de manter: se a formatação do CSV mudar, apenas o Agente 3 é alterado. Se quisermos extrair mais dados, alteramos apenas o Agente 2.

### B. `SerperDevTool` vs. Scraping Direto (A Decisão Crítica)

A decisão técnica mais importante foi **não fazer scraping direto** (com ferramentas como `BeautifulSoup` ou `Selenium`).

Sites de e-commerce como Amazon e Mercado Livre são ecossistemas complexos e hostis ao scraping:

* **Conteúdo Dinâmico:** Preços e disponibilidade são carregados via JavaScript, tornando o `Requests` + `BeautifulSoup` ineficazes.  
* **Bloqueio Ativo:** O uso de navegadores automatizados (`Selenium`) é rapidamente detectado, levando a CAPTCHAs, bloqueios de IP e mudanças frequentes de seletores HTML/CSS.

**Nossa Solução:** Usamos a `SerperDevTool` (uma API de resultados de busca do Google). Em vez de *visitar* a página, nós *perguntamos ao Google* sobre o produto em um site específico (ex: `Cartucho HP 664 site:amazon.com.br`).

**Vantagens:**

1. **Robustez:** A API sempre retorna um JSON estruturado. Não dependemos de seletores HTML que mudam.  
2. **Eficiência:** Evitamos o carregamento de navegadores pesados, CAPTCHAs e bloqueios.  
3. **Dados Pré-Extraídos:** A API do Google, muitas vezes, já identifica e retorna o **preço** e a **URL** em campos separados, facilitando o trabalho do `Agente Extrator`.

---

## 2. Estratégias Descartadas (O que não funcionou)

No desenvolvimento de soluções de scraping, o caminho feliz raramente é o primeiro.

* **Tentativa 1: `Requests` + `BeautifulSoup`:** Foi a primeira ideia. Falhou imediatamente, pois o HTML retornado não continha os dados de preço ou disponibilidade, que são carregados dinamicamente por JavaScript.  

* **Tentativa 2: `Selenium` / `Playwright`:** A próxima abordagem lógica seria automatizar um navegador real. Embora isso *tecnicamente* funcione para renderizar o JavaScript, foi descartado por ser:  
    * **Frágil:** A estrutura de classes CSS e IDs da Amazon, por exemplo, é ofuscada e muda constantemente. O script quebraria semanalmente.  
    * **Lento:** Iniciar um navegador para cada busca é computacionalmente caro.  
    * **Bloqueável:** Após algumas requisições, o site detectaria a automação e exigiria intervenção manual (CAPTCHA).  

* **Tentativa 3: Agente "Navegador" (Custom Tool):** Consideramos criar uma ferramenta personalizada para o `CrewAI` que usasse `Selenium` (ex: `browse_page(url)`). Isso se mostrou complexo demais para o agente gerenciar. O LLM teria dificuldade em "adivinhar" os seletores CSS corretos para clicar ou extrair, tornando a `SerperDevTool` uma abstração muito mais limpa e confiável.

---

## 3. Resultados e Próximos Passos

Estou muito satisfeito com o resultado alcançado. O código atual **cumpre seu objetivo principal com sucesso**: ele orquestra os agentes para buscar múltiplos produtos em múltiplos sites e gera um arquivo `.csv` limpo e estruturado com Nome, Preço, URL e Disponibilidade. A abordagem de usar a API de busca provou ser estável e eficaz.

### Melhorias Futuras

O sistema não é perfeito. A principal limitação da nossa abordagem atual é a dificuldade em capturar dados que **não** aparecem nos "snippets" de busca do Google. O exemplo mais claro é a **Avaliação (estrelas)** do produto.

A API `SerperDevTool` raramente retorna a contagem de estrelas nos resultados. Para obter esse dado de forma confiável, uma futura melhoria poderia ser:

1. **Manter a Busca:** O `Agente Pesquisador` continua usando a `SerperDevTool` para obter a URL final do produto (o que já faz bem).  
2. **Scraping Híbrido:** Criar uma nova ferramenta (ex: `get_page_content(url)`) que use uma biblioteca de scraping mais simples (como `BeautifulSoup`) para fazer uma única requisição HTTP *direcionada* àquela URL.  
3. **Extração Focada:** O `Agente Extrator` usaria o conteúdo dessa página *apenas* para encontrar a informação de avaliação, que é um alvo muito menor e mais fácil de extrair do que a página inteira.

Mesmo com essa melhoria pendente, a arquitetura atual é um excelente ponto de partida, equilibrando robustez e funcionalidade.
