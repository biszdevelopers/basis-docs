---
layout: home

hero:
  name: basis-auth
  text: The identity boundary for Basis applications
  tagline: An OpenID Connect provider that authenticates people through Microsoft Entra and issues tokens for registered applications and APIs.
  actions:
    - theme: brand
      text: Get Started
      link: /getting-started
    - theme: alt
      text: Architecture
      link: /architecture
    - theme: alt
      text: Integration Guide
      link: /wiring-up

features:
  - icon: 🔐
    title: Authorization code + PKCE
    details: The provider supports authorization code flow. S256 PKCE, state, nonce, and a registered redirect URI are required.
  - icon: 🪪
    title: App-specific access
    details: Each access token is issued for one registered resource audience. User permissions are stored per client application.
  - icon: 🗄️
    title: PostgreSQL-backed
    details: Users, organizations, client registrations, sessions, consents, permissions, codes, and refresh tokens are stored in PostgreSQL.
  - icon: 🧰
    title: Integration tools
    details: Use the client-registration CLI, a live registry doctor, a development-only OIDC demo, and a private internal API for trusted management integrations.
---
