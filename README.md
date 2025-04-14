# Manutenção e Evolução de Software - TP2

Nos últimos anos, os modelos de linguagem de grande escala (LLMs) têm demonstrado grande potencial em tarefas relacionadas ao processamento de linguagem natural. Apesar de seu uso crescente em diversos domínios da engenharia de software, ainda são escassas as avaliações sistemáticas e cientificamente embasadas de sua aplicação em contextos específicos. Este trabalho tem como objetivo realizar uma análise mais rigorosa da utilização de LLMs no contexto de Manutenção e Evolução de Software (MES), investigando suas contribuições, limitações e possíveis impactos na prática.

## Grupo:
* Vinicius Silva Gomes - Ciência da Computação
* Vitor Emanuel Ferreira Vital - Ciência da Computação

---

## Objetivos

O principal objetivo do projeto é investigar a aplicabilidade de modelos de linguagem de grande escala (LLMs) na detecção e correção de code smells em programas fonte. Code smells são indícios de problemas na estrutura do código que, embora não impeçam sua execução, podem dificultar sua manutenção, compreensão e evolução ao longo do tempo. Dada a relevância desses aspectos no contexto de Manutenção e Evolução de Software (MES), explorar ferramentas automatizadas para auxiliar nessa tarefa é de grande importância prática e científica.

Mais especificamente, este trabalho busca avaliar se LLMs são capazes de identificar trechos de código que apresentam code smells, analisando sua sensibilidade a diferentes padrões e contextos de implementação. A partir de entradas compostas por trechos de código potencialmente mal estruturado, pretende-se observar se o modelo consegue detectar a presença de um problema, mesmo sem instruções explícitas ou exemplos prévios.

Outro objetivo central da pesquisa é verificar a capacidade de classificação dos code smells identificados. Ou seja, não basta que o modelo de linguagem aponte que há um problema no código: é igualmente importante que ele consiga nomear e categorizar corretamente o tipo de code smell presente — como por exemplo, Long Method, God Class, Feature Envy, entre outros. Essa etapa é essencial para validar o entendimento semântico do modelo sobre boas práticas de design e arquitetura de software.

Por fim, a pesquisa também pretende explorar a capacidade dos LLMs de sugerirem correções ou refatorações adequadas para os problemas identificados. Avaliar a qualidade dessas sugestões, tanto do ponto de vista técnico quanto em relação à aderência a princípios de engenharia de software, permitirá compreender o real potencial desses modelos como ferramentas auxiliares no processo de manutenção e melhoria contínua do código.

## Metodologia

### LLM utilizado

Escolhemos o modelo DeepSeek como o foco da análise para as tarefas de detecção, classificação e correção de code smells. Essa escolha se justifica pelo interesse em investigar o desempenho de um modelo recente, de código aberto, que foi treinado com uma quantidade menor de recursos computacionais e dados em comparação com modelos consolidados como o ChatGPT e o LLaMA, da Meta. A proposta é avaliar até que ponto um modelo mais leve e acessível consegue entregar resultados competitivos em tarefas de engenharia de software, especialmente em contextos de Manutenção e Evolução de Software (MES).

Modelos como GPT-4 e LLaMA já foram amplamente estudados em pesquisas envolvendo atividades como identificação de code smells, geração de código, e sugestões de refatoração. Eles se beneficiam de arquiteturas altamente otimizadas e treinamento em larga escala, o que naturalmente os coloca em vantagem em termos de capacidade geral de raciocínio e compreensão semântica. No entanto, o acesso a esses modelos muitas vezes é limitado, seja por barreiras financeiras, seja por restrições de uso em ambientes específicos.

Nesse contexto, o DeepSeek surge como uma alternativa promissora: por ser um modelo recente, com foco em código e linguagem natural, e de código aberto, ele representa uma opção mais viável para pesquisadores, desenvolvedores independentes e organizações com infraestrutura mais modesta. Avaliar sua eficácia em tarefas práticas, como a identificação e correção de code smells, permite não apenas comparar seu desempenho com o de modelos mais robustos, mas também entender o custo-benefício envolvido no uso de soluções mais acessíveis.

### Datasets

#### MLCQ: Industry-relevant code smell data set
O principal dataset utilizado na análise será o MLCQ. Ele foi escolhido entre várias opções por ser um dos mais confiáveis e bem organizados quando se trata de identificar e classificar code smells. Suas anotações seguem critérios claros e consistentes, o que dá mais segurança para a análise. Como o conjunto de dados é bastante grande, usaremos apenas uma parte dele nas nossas avaliações, o que vai permitir fazer inspeções manuais com mais cuidado e garantir que os resultados obtidos sejam relevantes e bem analisados.

#### Exemplos de códigos com code smell
Além do dataset anterior, iremos selecionar alguns repositórios de código públicos com alta cobertura de teste e iremos modificar alguns trechos para inserir manualmente code smells conhecidos. A ideia é, então, verificar se o modelo de linguagem é capaz de identificar o code smell e comparar o quão distante está a sugestão de modificação feita pelo modelo com relação a implementação base, antes da inserção manual do code smell.

### Exemplos preliminares de prompts

Para avaliar o desempenho do modelo na tarefa de detecção e classificação de code smells, serão utilizados prompts projetados para simular instruções realistas que um desenvolvedor ou ferramenta de apoio poderia fornecer. Esses prompts buscam avaliar a capacidade do modelo de identificar que há um problema no código e sua competência em reconhecer qual tipo específico de code smell está presente e, por fim, sugerir possíveis melhorias. Um exemplo de prompt que será utilizado no experimento seria:

> "Analyze the following code snippet and identify the most prominent code smell according to established software engineering principles. If there is no discernible code smell, respond with 'None'. Then, suggest which modifications or refactorings could be applied to mitigate the identified code smell."

Outros tipos de prompts serão utilizados ao longo da pesquisa. No entanto, a ideia é manter o foco em avaliar a capacidade de identificar os code smells e sugerir as mudanças e refatorações adequadas.

### Avaliação

#### Avaliação Quantitativa

A avaliação quantitativa do modelo DeepSeek será realizada com o objetivo de analisar sua performance em dois aspectos principais: a detecção e a correção de code smells. Essas avaliações serão conduzidas separadamente em cenários distintos.

No primeiro cenário, verificaremos a capacidade do modelo em identificar corretamente a presença ou ausência de code smells em trechos de código, bem como classificá-los de acordo com seu tipo. Para isso, será utilizado um conjunto de exemplos previamente rotulados, permitindo a comparação direta entre as saídas do modelo e os rótulos de referência. 

As métricas adotadas para essa avaliação serão acurácia, precisão, recall e F1-score.

O segundo cenário avaliaremos a capacidade do modelo de localizar com precisão o trecho de código afetado pelo code smell e, adicionalmente, sugerir uma correção apropriada. Essa etapa será realizada com base em códigos-fonte originalmente bem estruturados, nos quais serão introduzidos code smells artificialmente, de forma manual

As sugestões geradas pelo modelo serão comparadas ao código original e ao código modificado, por fim o trecho sugerido será incluído no código original a fim verificar se a alteração recomendada de fato mantém o comportamento do código.

Nesse contexto, será avaliado se os code smells indicados pelo modelo estão de fato presentes no código, se os code smells existentes foram corretamente corrigidos e, por fim, se o código resultante está correto, ou seja, mantendo o comportamento original do sistema.

Devido a complexidade de avaliação deste segundo cenário, iremos utilizar um número de exemplos reduzidos, já que a avaliação será feita pelos integrantes do grupo.

#### Avaliação Qualitativa

A avaliação qualitativa será conduzida com o intuito de compreender a utilidade prática, a clareza e a confiabilidade das sugestões fornecidas pelo modelo.

Essa análise permitirá obter uma visão mais rica e interpretativa sobre a performance do modelo, complementando os dados objetivos da avaliação quantitativa e ajudando a identificar oportunidades de aprimoramento tanto na detecção quanto na correção de code smells. Para tanto, iremos inspecionar minuciosamente os resultados obtidos e avaliar se, de modo geral, o DeepSeek realiza um trabalho satisfatório ao identificar e corrigir code smells.


