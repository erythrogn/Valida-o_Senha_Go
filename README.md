# Auth Package

Um pacote Go focado em segurança para autenticação, fornecendo hash seguro de senhas (bcrypt), emissão e validação estrita de tokens JWT (HS256) e aplicação rigorosa de políticas de senhas.

Este pacote foi desenhado para ser **livre de estado global** e **independente de variáveis de ambiente**, exigindo que toda a configuração (segredos, fator de trabalho, relógio) seja injetada explicitamente.

## 🛡️ Principais Funcionalidades de Segurança

* **Proteção JWT Estrita:**
* Bloqueio ativo contra ataques de `alg: none` e alteração de algoritmo.
* Exigência das claims de tempo (`exp`, `nbf`, `iat`) validadas por um relógio injetável.
* Geração automática de `jti` (JWT ID) usando entropia forte (`crypto/rand`).
* Tolerância a dessincronização de relógios (`leeway`) limitada e segura.


* **Hashing Resiliente (bcrypt):**
* Defesa contra truncamento silencioso (senhas limitadas a 72 bytes antes do hash).
* Custo mínimo obrigatório (fator de trabalho `12`) para resistir a ataques de força bruta modernos.
* Mitigação de ataques de temporização (*Timing Attacks*) via função `DummyHash`.


* **Higiene e Complexidade de Senhas:**
* Mínimo de 12 caracteres.
* Exigência de letras maiúsculas, minúsculas, números e símbolos.
* Bloqueio de caracteres de controle e espaços periféricos.
* Rejeição de senhas que contenham dados do próprio usuário (nome, e-mail).
* Suporte a injeção de *Denylists* (ex: integração com *Have I Been Pwned*).


* **Abstração de Erros:**
* Uso exclusivo de erros sentinela (`errors.Is`), ocultando regras internas e prevenindo o vazamento de informações para atacantes.



---

## 🚀 Como Usar

### 1. Inicializando o Gerenciador JWT

A configuração deve ser carregada pela sua infraestrutura e passada explicitamente para o pacote.

```go
package main

import (
	"log"
	"time"
	"seuprojeto/auth"
)

func main() {
	cfg := auth.TokenConfig{
		Secret:   []byte("sua-chave-super-secreta-com-pelo-menos-32-bytes!"),
		Issuer:   "sua-api.empresa.com",
		Audience: "portal-web",
		TTL:      30 * time.Minute,
		Leeway:   5 * time.Second,
		Clock:    time.Now,
	}

	tokenMgr, err := auth.NewTokenManager(cfg)
	if err != nil {
		log.Fatalf("falha ao inicializar o gerenciador JWT: %v", err)
	}
	
	// tokenMgr agora pode ser injetado nos seus handlers
}

```

### 2. Validação Forte de Senhas

Antes de aceitar um cadastro ou troca de senha, verifique a força da credencial. Você pode passar dados do usuário para garantir que ele não usou o próprio nome como senha.

```go
senhaDesejada := "MinhaS3nh@Forte!2026"
nome := "Joao"
email := "joao@email.com"

// O segundo parâmetro (nil) aceita uma interface Denylist customizada.
err := auth.CheckStrength(senhaDesejada, nil, nome, email)
if err != nil {
    // Retornar err.Error() de forma segura, os erros são genéricos.
    fmt.Println("Erro de validação:", err)
}

```

### 3. Hash e Verificação (com mitigação de Timing Attacks)

```go
// Gerando um hash para salvar no banco (custo recomendado: 12)
hash, err := auth.Hash(senhaDesejada, 12)
if err != nil {
    log.Fatal(err)
}

// ---------------------------------------------------------
// Fluxo de Login Seguro:

hashDoDB := buscarHashNoBanco(email)

if hashDoDB == "" {
    // O usuário não existe. Rodamos o DummyHash para gastar a mesma
    // quantidade de tempo de CPU que gastaríamos validando um usuário real.
    _ = auth.DummyHash(12)
    return errors.New("credenciais inválidas")
}

// Verificando a senha
err = auth.Verify(hashDoDB, senhaTentativa)
if err != nil {
    // Retornará auth.ErrInvalidCredentials (não vaza detalhes)
    return errors.New("credenciais inválidas")
}

```

### 4. Emissão e Validação de Tokens

```go
// Emissão
tokenStr, err := tokenMgr.Generate(123, "joao@email.com", "Joao")
if err != nil {
    log.Fatal(err)
}

// Validação em middlewares
claims, err := tokenMgr.Validate(tokenStr)
if err != nil {
    if errors.Is(err, auth.ErrTokenExpired) {
        fmt.Println("Token expirado, faça login novamente.")
        return
    }
    fmt.Println("Token inválido ou corrompido.")
    return
}

fmt.Printf("Acesso autorizado para o usuário %d (%s)\n", claims.UserID, claims.FirstName)

```

---

## ⚠️ Responsabilidades da Aplicação Consumidora

Embora este pacote blinde a lógica de autenticação, o ecossistema externo **precisa** garantir os seguintes controles:

1. **Gestão de Segredos:** O `TokenConfig.Secret` deve vir de um cofre (ex: HashiCorp Vault, AWS Secrets Manager) ou variável de ambiente lida apenas na inicialização (`main.go`). **Nunca fixe (hardcode) a chave no código.**
2. **Rate Limiting:** Por usar o algoritmo `bcrypt` intencionalmente pesado, endpoints de `/login` e `/register` devem ser protegidos por limitadores de taxa (Rate Limit) baseados em IP ou conta para evitar ataques de esgotamento de recursos (DoS).
3. **Transporte Seguro (TLS):** Todos os tokens e senhas devem trafegar estritamente sob **HTTPS**.
4. **Armazenamento no Cliente:** Se utilizar navegadores, devolva o JWT dentro de cookies configurados com `HttpOnly=true`, `Secure=true` e `SameSite=Lax` (ou `Strict`). Não armazene tokens no `localStorage`.
5. **Revogação (Logout):** Como os JWTs são auto-contidos, para invalidá-los antes da expiração, a aplicação deve manter uma *Blocklist* (ex: Redis) armazenando o ID do token (`claims.ID`) até o seu TTL acabar.
