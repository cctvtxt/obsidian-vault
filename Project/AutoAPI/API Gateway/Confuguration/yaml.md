```yaml
apiVersion: gateway.mycorp.io/v1alpha1
kind: GatewayConfig
metadata:
  name: public-api-gateway
  revision: "2026-04-19.002"

spec:
  server:
    publicBaseUrl: https://api.example.com
    listen:
      port: 3000
      globalPrefix: /api
    trustProxy: true

  docs:
    enabled: true
    route:
      uiPath: /docs
      jsonPath: /docs-json
    source: gateway-runtime
    openapi:
      title: Public API Gateway
      description: Unified gateway API for auth and platform endpoints
      version: 1.0.0
      tags:
        - name: auth
          description: Authentication endpoints
      securitySchemes:
        bearer:
          type: http
          scheme: bearer
          bearerFormat: JWT
    exposure:
      includeRoutes:
        - auth-login
        - auth-me
      hideInternalRoutes: true
    ui:
      persistAuthorization: true
      displayRequestDuration: true
      docExpansion: list
      tryItOutEnabled: true
    cache:
      enabled: true
      ttl: 60s

  observability:
    requestIdHeader: x-request-id
    correlationIdHeader: x-correlation-id
    tracing:
      enabled: true
      propagate:
        - x-request-id
        - x-correlation-id
        - traceparent
        - tracestate

  security:
    cors:
      enabled: true
      allowOrigins:
        - https://app.example.com
      allowMethods: [GET, POST, OPTIONS]
      allowHeaders:
        - authorization
        - content-type
        - x-request-id
        - x-correlation-id
        - x-tenant-id
      exposeHeaders:
        - x-request-id
      allowCredentials: true
      maxAgeSeconds: 600

    headers:
      forward:
        - x-request-id
        - x-correlation-id
        - x-forwarded-for
        - x-forwarded-proto
        - x-real-ip
        - user-agent
        - x-tenant-id
      drop:
        - x-internal-debug
        - x-service-token

  upstreamDefaults:
    protocol: grpc
    discovery: kubernetes-dns
    connectTimeout: 2s
    requestTimeout: 5s
    idleTimeout: 30s
    retries:
      enabled: true
      retryableStatuses:
        - UNAVAILABLE
        - DEADLINE_EXCEEDED
      attempts: 1
      perTryTimeout: 1500ms
    circuitBreaker:
      enabled: true
      maxConnections: 1000
      maxPendingRequests: 500
      maxRequests: 5000
    outlierDetection:
      enabled: true
      consecutive5xx: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50

  services:
    - id: auth-grpc
      kind: grpc
      kubernetes:
        namespace: auth
        serviceName: auth-svc
        dns: auth-svc.auth.svc.cluster.local
        port: 9090
      grpc:
        authority: auth-svc.auth.svc.cluster.local
        package: auth.v1
      healthCheck:
        enabled: true
        type: grpc
        service: grpc.health.v1.Health

  authProviders:
    - id: jwt-access
      type: jwt
      fromHeader: Authorization
      scheme: Bearer
      jwksUri: https://identity.example.com/.well-known/jwks.json
      cacheTtl: 10m
      clockSkew: 30s
      requiredClaims:
        - sub
        - exp

  routes:
    - id: auth-login
      enabled: true
      type: proxy
      match:
        path: /api/auth/login
        method: POST
      docs:
        enabled: true
        tag: auth
        summary: Login
        description: Authenticates user and returns tokens
        requestBodySchema: AuthLoginRequest
        responseSchema:
          "200": AuthLoginResponse
          "400": ErrorResponse
          "401": ErrorResponse
          "429": ErrorResponse
      auth:
        mode: public
      validation:
        contentTypes:
          - application/json
        maxBodyBytes: 32768
      rateLimit:
        - key: ip
          limit: 20
          window: 1m
        - key: ip+path
          limit: 100
          window: 10m
      upstream:
        serviceRef: auth-grpc
        method: auth.v1.AuthService/Login
        timeout: 3s
        retries:
          enabled: false
        requestMapping:
          transport: grpc
          body: "*"
          headersToMetadata:
            x-request-id: x-request-id
            x-correlation-id: x-correlation-id
            x-tenant-id: x-tenant-id
            user-agent: user-agent
            x-forwarded-for: client-ip
        responseMapping:
          setHeaders:
            cache-control: no-store
      audit:
        enabled: true
        event: auth.login.attempt
        redactFields:
          - password
          - otp

    - id: auth-me
      enabled: true
      type: proxy
      match:
        path: /api/auth/me
        method: GET
      docs:
        enabled: true
        tag: auth
        summary: Current user
        description: Returns current authenticated user profile
        security:
          - bearer: []
        responseSchema:
          "200": AuthMeResponse
          "401": ErrorResponse
          "403": ErrorResponse
      auth:
        mode: jwt
        providerRef: jwt-access
        requiredScopes:
          - profile:read
      rateLimit:
        - key: subject
          limit: 120
          window: 1m
        - key: tenant
          limit: 3000
          window: 1m
      upstream:
        serviceRef: auth-grpc
        method: auth.v1.AuthService/GetMe
        timeout: 2s
        retries:
          enabled: true
          retryableStatuses:
            - UNAVAILABLE
            - DEADLINE_EXCEEDED
          attempts: 1
          perTryTimeout: 1500ms
        requestMapping:
          transport: grpc
          headersToMetadata:
            x-request-id: x-request-id
            x-correlation-id: x-correlation-id
            x-tenant-id: x-tenant-id
          claimsToMetadata:
            sub: x-auth-sub
            tenant_id: x-auth-tenant-id
            scope: x-auth-scope
            roles: x-auth-roles
        responseMapping:
          setHeaders:
            cache-control: no-store, private
      audit:
        enabled: true
        event: auth.me.read

    - id: docs-ui
      enabled: true
      type: local
      match:
        path: /api/docs
        method: GET
      handler:
        kind: swagger-ui
      auth:
        mode: public
      rateLimit:
        - key: ip
          limit: 60
          window: 1m

    - id: docs-json
      enabled: true
      type: local
      match:
        path: /api/docs-json
        method: GET
      handler:
        kind: swagger-json
      auth:
        mode: public
      rateLimit:
        - key: ip
          limit: 120
          window: 1m

  schemas:
    AuthLoginRequest:
      type: object
      required: [login, password]
      properties:
        login:
          type: string
          example: user@example.com
        password:
          type: string
          format: password
        otp:
          type: string
          nullable: true
        deviceId:
          type: string
          nullable: true

    AuthLoginResponse:
      type: object
      properties:
        accessToken:
          type: string
        refreshToken:
          type: string
        expiresIn:
          type: integer
        user:
          $ref: "#/components/schemas/UserProfile"

    AuthMeResponse:
      type: object
      properties:
        user:
          $ref: "#/components/schemas/UserProfile"

    UserProfile:
      type: object
      properties:
        id:
          type: string
        email:
          type: string
        displayName:
          type: string

    ErrorResponse:
      type: object
      properties:
        code:
          type: string
        message:
          type: string
        requestId:
          type: string
```