# Pseudocódigo do pipeline

Parâmetros sem valor são definidos por calibração ou medição; `MAX_INTENCOES = 2` é decisão de projeto.

```
PARÂMETROS  (valores definidos por calibração ou medição, não fixados neste documento)
  MODELO_LLM, MODELO_EMBEDDING
  TOP_K, CORTE_MINIMO, TEMPO_LIMITE, TAMANHO_MAX_MENSAGEM
  NUM_CTX                 // fixado explicitamente; não depender do padrão do Ollama
  MAX_INTENCOES = 2
  LISTA_ALTO_RISCO = termos de ameaça jurídica, reembolso, cobrança duplicada,
                     pedido de atendente, entre outros

BASE (preparada uma vez, antes de atender)
  para cada item do FAQ: pergunta, resposta, categoria, revisão (12/2025)
  nota interna original → reescrita como conteúdo para o cliente
                          OU como instrução ao modelo (revisada pelo suporte)
  vetor indexado: pergunta + resposta (a comparar com só a pergunta)

FUNÇÃO atender(mensagem):
  TENTAR

    // 1. Validação básica
    SE mensagem está vazia OU excede TAMANHO_MAX_MENSAGEM:
        RETORNAR escalar(mensagem, motivo = "mensagem inválida")

    // 2. Rede de segurança por palavras: só pode escalar, nunca responder
    SE mensagem contém algum termo de LISTA_ALTO_RISCO:
        RETORNAR escalar(mensagem, motivo = "termo de alto risco")

    // 3. Classificação de intenção (LLM local, temperatura baixa, saída estruturada)
    //    a mensagem do cliente entra no prompt delimitada como DADO, não como instrução
    //    "está fora do ar?" → categoria Status do sistema (segue o FAQ, página de status);
    //    só "requer humano" se o cliente disser que afeta todos ou que a página não mostra nada
    resultado ← classificar(mensagem)
    SE resultado.requer_humano
       OU resultado.fora_de_escopo
       OU quantidade(resultado.intencoes) > MAX_INTENCOES:
        RETORNAR escalar(mensagem, motivo = resultado.motivo)

    // 4. Recuperação, uma busca por intenção
    PARA CADA intencao EM resultado.intencoes:
        vetor ← gerar_embedding(intencao.trecho_da_mensagem, MODELO_EMBEDDING)
        trechos[intencao] ← buscar_top_k(vetor, TOP_K)
        // conferir se o banco devolve distância (menor é melhor)
        // ou similaridade (maior é melhor) antes de comparar
        SE melhor_score(trechos[intencao]) NÃO atinge CORTE_MINIMO:
            RETORNAR escalar(mensagem, motivo = "sem evidência suficiente")
        registrar_margem(trechos[intencao])      // apenas registro, não bloqueia

    // 5. Geração fundamentada
    contexto ← unir(trechos de todas as intenções)   // resposta + nota reescrita
    SE tamanho_em_tokens(prompt montado) > NUM_CTX:
        RETORNAR escalar(mensagem, motivo = "prompt excede o contexto")
    saida ← gerar_resposta(mensagem, contexto)
            // somente o contexto, temperatura baixa, mensagem delimitada como dado,
            // dados da conta sempre de forma condicional ("se o seu plano for...")
            // saída estruturada: texto, ids_dos_trechos_usados, contexto_suficiente

    // 6. Verificação em cascata: qualquer falha escala a mensagem inteira
    SE saida.contexto_suficiente = NÃO:
        RETORNAR escalar(mensagem, motivo = "modelo declarou contexto insuficiente")
    SE saida.ids_usados contém id que NÃO está em contexto:
        RETORNAR escalar(mensagem, motivo = "citou trecho não recuperado")
    SE NÃO resposta_fundamentada(saida.texto, contexto, mensagem):
        // verifica (1) sustentado pelos trechos E (2) responde ao que foi perguntado
        // implementação com ou sem LLM: decidida por medição
        RETORNAR escalar(mensagem, motivo = "resposta não sustentada ou não responde")
    PARA CADA intencao EM resultado.intencoes:
        SE categoria dos trechos usados DIVERGE de intencao.categoria:
            RETORNAR escalar(mensagem, motivo = "categoria divergente")
    SE saida.texto reproduz nota interna ou instrução ao modelo:
        RETORNAR escalar(mensagem, motivo = "possível vazamento de nota interna")

    // 7. Resposta automática
    texto_final ← saida.texto + AVISO_PADRAO_COMO_FALAR_COM_ATENDENTE
    registrar(mensagem, intencoes, trechos, scores, saida, decisao = "respondeu")
    RETORNAR responder(texto_final)

  EM CASO DE ERRO TÉCNICO OU TEMPO_LIMITE EXCEDIDO:
    alertar_localmente("falha técnica no pipeline")   // sem dados de cliente
    RETORNAR escalar(mensagem, motivo = "falha técnica")


FUNÇÃO escalar(mensagem, motivo):            // NÃO depende do LLM
    pacote ← { mensagem original, categorias, trechos recuperados,
               rascunho de resposta (se existir, marcado como NÃO verificado),
               motivo do escalonamento, horário }
    enfileirar_para_atendente(pacote)
    registrar(pacote, decisao = "escalou")
    RETORNAR aviso_ao_cliente("Um atendente vai dar continuidade ao seu atendimento.")
```
