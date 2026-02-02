# **SAPI, Error Handling, Conteúdo e testes**

## 

## Objetivo:
Ter respostas consistentes em JSON, com httpStatus correto, correlationId, e header x-correlation-id.

##### Estrutura padrão de resposta de erro

{
"success": false,
"error": {
"httpStatus": 400,
"errorType": "APIKIT:BAD\_REQUEST",
"message": "Mensagem do erro",
"correlationId": "...."
}
}

###### Parte A, Preparação no Listener da SAPI

1. No flow main, no HTTP Listener, confirmar que o Response usa:
   1.1) Status code: #\[vars.httpStatus default 200]
   1.2) Headers: #\[vars.outboundHeaders default {}]
   Isto garante que o status e headers que defines no error handler são mesmo devolvidos ao cliente.
   
2. Em cada On Error Propagate, vamos:
   2.1) vars.httpStatus com o código certo
   2.2) payload com o JSON de erro
   2.3) vars.outboundHeaders com Content-Type e x-correlation-id

###### Parte B, Headers padrão para todos os handlers

**Set Variable**
Name: outboundHeaders
Value:
#\[{
"Content-Type": "application/json",
"x-correlation-id": correlationId
}]

**Nota:**
Usar correlationId sem parênteses.

###### Parte C, DataWeave padrão para todos os handlers

**Transform Message, Output Payload:**

%dw 2.0
output application/json

{
success: false,
error: {
httpStatus: vars.httpStatus,
errorType: (error.errorType as String),
message: (error.description default "Unexpected error."),
correlationId: correlationId
}
}

###### Parte D, Mapeamento do APIkit

1. On Error Propagate type: APIKIT:BAD\_REQUEST
   Set Variable httpStatus = 400
   Depois Transform Message com o script padrão
   Depois Set Variable outboundHeaders com o header padrão

Porque
Pedido inválido, por exemplo falta de campo obrigatório, path param inválido, query param inválido.

2. On Error Propagate type: APIKIT:NOT\_FOUND
   Set Variable httpStatus = 404
   Transform padrão
   Headers padrão

Porque
Rota não existe no RAML, ou path errado.

3. On Error Propagate type: APIKIT:METHOD\_NOT\_ALLOWED
   Set Variable httpStatus = 405
   Transform padrão
   Headers padrão

Porque
A rota existe, mas o método não existe para essa rota, por exemplo POST onde só há GET.

4. On Error Propagate type: APIKIT:NOT\_ACCEPTABLE
   Set Variable httpStatus = 406
   Transform padrão
   Headers padrão

Porque
O Accept do cliente não é compatível com o que a API devolve.

5. On Error Propagate type: APIKIT:UNSUPPORTED\_MEDIA\_TYPE
   Set Variable httpStatus = 415
   Transform padrão
   Headers padrão

Porque
Content-Type do request não é suportado, por exemplo mandar text/plain quando o RAML pede application/json.

6. On Error Propagate type: APIKIT:NOT\_IMPLEMENTED
   Set Variable httpStatus = 501
   Transform padrão
   Headers padrão

Porque
Rota está no RAML mas não está implementada corretamente, ou falta mapping no router.

###### Parte E, Erros da lógica na implementação

1. On Error Propagate type: APP:NOT\_FOUND
   Set Variable httpStatus = 404
   Transform padrão
   Headers padrão

Porque
Se decide que um recurso não existe, por exemplo select à BD devolve vazio, e faz-se Raise Error APP:NOT\_FOUND.

2. On Error Propagate type: DB:\*
   Set Variable httpStatus = 500
   Transform padrão
   Headers padrão

Porque
Falha técnica no acesso à BD, por exemplo credenciais, timeout, SQL inválido.

**Nota prática:
Pode-se usar 503 quando a BD está indisponível, mas 500 é aceitável para o trabalho e é simples.

###### Parte F, Como testar no Postman, SAPI

Base URL
SAPI, porta 8083
http://localhost:8083

**Dica:**
Em cada request, meter header:
x-correlation-id: test-001

Assim confirma-se se a resposta devolve o mesmo correlationId ou se usa o gerado pelo Mule.

# Testes APIkit

1. METHOD\_NOT\_ALLOWED
   Faz POST numa rota que só tem GET.

   **Exemplo**
POST http://localhost:8083/api/account

   Se no RAML só existir GET para essa rota, deve devolver 405.
   
2. UNSUPPORTED\_MEDIA\_TYPE
   Faz POST com Content-Type errado.

   POST http://localhost:8083/api/account
   **Headers**
Content-Type: text/plain
   **Body**
abc
   Deve devolver 415.
   
3. NOT\_ACCEPTABLE
   Mete Accept inválido.

   GET http://localhost:8083/api/account
   **Headers**
Accept: application/xml
   Deve devolver 406, se a API só devolver application/json.
   
4. BAD\_REQUEST
   Depende do RAML, mas consegue-se forçar BAD\_REQUEST assim:
   Chamar um endpoint com path param obrigatório em falta.

   **Exemplo:**
   GET http://localhost:8083/api/account/
   Ou chamar com param no formato errado se o RAML tiver pattern, por exemplo accountId numérico.
   
5. APIKIT NOT\_FOUND
   Rota inexistente

    GET http://localhost:8083/api/istoNaoExiste
    Deve devolver 404.

# Testes da implementação

1. APP NOT\_FOUND em GET por ID
   GET http://localhost:8083/api/account/A99999
   Usa um accountId que não exista na BD.

   O flow deve fazer select, ficar vazio, e dar Raise Error APP:NOT\_FOUND.
   Resultado esperado 404, JSON padrão.
   
2. DB:\* falha de BD
   Forma simples
   Desliga o MySQL ou muda temporariamente a password do Database Config, faz redeploy, e chama:

GET http://localhost:8083/api/account
Deve devolver 500, JSON padrão, errorType DB:....

# Testes de consistência
Em todos os casos acima confirmar:

1. Status code no Postman é o que se definiu em vars.httpStatus   
2. Body tem success false e error.httpStatus igual ao status HTTP   
3. Header x-correlation-id aparece na resposta   
4. correlationId no body existe

