# Autenticação — OAuth2 e JWT

Visão prática sobre padrões modernos de autenticação e autorização: OAuth2 para delegação e JWT para tokens compactos.

## OAuth2 (fluxos comuns)

- **Authorization Code**: fluxo recomendado para apps web e mobile com servidor (uso de PKCE em clientes públicos).
- **Client Credentials**: usado para comunicação servidor-a-servidor.
- **Implicit**: obsoleto; não usar.

Fluxo Authorization Code (resumido):

1. Cliente redireciona usuário ao Authorization Server (AS) com `client_id`, `redirect_uri`, `scope`.
2. Usuário autentica e autoriza; AS retorna `code` ao `redirect_uri`.
3. Cliente troca `code` por `access_token` (e opcionalmente `refresh_token`) no token endpoint.

## JWT (JSON Web Tokens)

- JWTs são tokens compactos contendo `header.payload.signature`.
- Use JWTs assinados (HS256 ou RS256). Prefira algoritmos assimétricos (RS256) para maior segurança entre serviços.
- Evite colocar dados sensíveis no payload; trate como dados NÃO confiáveis.

Exemplo simples de verificação (pyjwt):

```python
import jwt

token = 'ey...'
payload = jwt.decode(token, public_key, algorithms=['RS256'])
```

## Refresh tokens e logout

- Use `refresh_token` para renovar `access_token` sem reautenticação do usuário.
- Para logout, invalide `refresh_token` no servidor; para JWTs curtos, use short-lived access tokens.

## Boas práticas

- Nunca confie em JWT sem verificar assinatura e `exp` (expiration).
- Use HTTPS sempre; não transporte tokens em query strings.
- Centralize validação de tokens (gateway, middleware) para evitar duplicação.
- Considere revogação via blacklist/registry se necessário (trade-offs de performance).

Referências: RFC 6749 (OAuth2), RFC 7519 (JWT), OpenID Connect.
