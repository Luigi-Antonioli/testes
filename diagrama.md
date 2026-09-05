# Diagrama de Sequência: Realizar Venda (Com Pagamentos e Recursos Offline/Inteligentes)

```mermaid
sequenceDiagram
    actor Vendedor
    participant Tela as Tela Nova Venda
    participant Catalogo as Catálogo (UC-12)
    participant Carrinho
    participant Estoque
    participant Pagamento as Pagamento (UC-05)
    participant Sincronizador as Sincronizador Local / DB

    Vendedor ->> Tela: acionar "Nova Venda"
    Tela -->> Vendedor: exibir campo de busca
    Vendedor ->> Tela: digitar termo do produto
    Tela ->> Catalogo: buscarProdutos(termo)
    Catalogo -->> Tela: lista de produtos
    Tela -->> Vendedor: exibir resultados
    Vendedor ->> Tela: selecionar produto
    Tela -->> Vendedor: pedir quantidade
    Vendedor ->> Tela: informar quantidade
    
    Tela ->> Estoque: consultarEstoque(produtoId, qtd)
    Estoque -->> Tela: disponível? (true/false, saldo, alertaEstoqueMinimo)

    opt Alerta de Estoque Mínimo
        Tela -->> Vendedor: exibir alerta "Estoque próximo do nível mínimo"
    end

    alt estoque suficiente
        Tela ->> Carrinho: adicionarItem(produto, qtd)
        Carrinho ->> Carrinho: recalcularTotal()
        
        Note right of Tela: Sugerir produtos em conjunto<br/>com base no histórico
        Tela ->> Catalogo: consultarSugestoesProdutos(produtoId)
        Catalogo -->> Tela: listaProdutosSugeridos
        
        Carrinho -->> Tela: total atualizado + sugestões de produtos
        Tela -->> Vendedor: item adicionado, exibir carrinho e sugestões

        opt adicionar mais itens
            Note right of Vendedor: Vendedor repete os passos de busca/seleção<br/>(loop sobre passos 2 a 5 do UC-02)
        end

        Vendedor ->> Tela: confirmar carrinho
        Tela -->> Vendedor: exibir resumo (itens, qtds, total)

        opt vendedor remove item do carrinho ou cancela venda
            Tela -->> Vendedor: solicitar confirmação de ação crítica
            Vendedor ->> Tela: confirmar cancelamento/remoção
            
            alt se remoção de item
                Tela ->> Carrinho: removerItem(itemId)
                Carrinho ->> Carrinho: recalcularTotal()
                Carrinho -->> Tela: total atualizado
                Tela -->> Vendedor: exibir carrinho recalculado
            end
        end

        Vendedor ->> Tela: avançar para pagamento
        
        Note right of Tela: [R52, R58] Validar vínculo do operador<br/>com iniciativa cadastrada
        Tela ->> Pagamento: validarVinculoEStatusOperador(vendedorId)

        alt Vínculo ativo e válido
            Pagamento -->> Tela: vinculoAtivo (iniciativaId, contaRecebimentoValida)
            
            Note right of Tela: [R53, R55, R56] Iniciar pagamento com rastreabilidade
            Tela ->> Pagamento: iniciarPagamento(total, itens, vendedorId, iniciativaId, chavePix/conta)
            Pagamento ->> Pagamento: registrarIdentificacaoOperador(vendedorId, "CRIACAO")
            
            alt Com Conexão com a Internet
                Pagamento -->> Tela: redirecionar para fluxo de pagamento online
            else Sem Conexão (Modo Offline)
                Pagamento ->> Sincronizador: registrarVendaELocalmente(vendaData)
                Sincronizador ->> Estoque: atualizarEstoqueLocal(itens)
                Pagamento -->> Tela: venda salva offline com sucesso
                Tela -->> Vendedor: confirmar venda e informar "Será sincronizado ao reconectar"
            end

        else Vínculo cancelado, expirado ou ausente [R58]
            Pagamento -->> Tela: erroVinculoInvalidoOuExpirado
            Tela -->> Vendedor: bloquear cobrança, exibir alerta (vínculo expirado/inválido)
        end

    else produto sem estoque - A1
        Estoque -->> Tela: estoqueEsgotado(produtoId)
        Tela -->> Vendedor: bloquear adição, exibir alerta
        Note right of Vendedor: Vendedor seleciona outro produto<br/>ou cancela a venda
    end

    Note right of Estoque: RB-02: Nenhuma operação pode<br/>resultar em saldo negativo no estoque
