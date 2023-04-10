# Configurando autenticação no Backstage via keycloak sem a necessidade de um proxy


## Edite o arquivo packages/app/src/apis.ts

Aqui esta o arquivo completo


import {
  ScmIntegrationsApi,
  scmIntegrationsApiRef,
  ScmAuth,
} from '@backstage/integration-react';
import {
  AnyApiFactory,
  ApiRef,
  BackstageIdentityApi,
  configApiRef,
  createApiFactory,
  createApiRef,
  discoveryApiRef,
  OAuthApi,
  oauthRequestApiRef,
  OpenIdConnectApi,
  ProfileInfoApi,
  SessionApi,
} from '@backstage/core-plugin-api';
import { OAuth2 } from '@backstage/core-app-api';

export const keycloakAuthApiRef: ApiRef<
  OAuthApi &
	OpenIdConnectApi &
	ProfileInfoApi &
	BackstageIdentityApi &
	SessionApi
> = createApiRef({
  id: 'auth.keycloak',
});

export const apis: AnyApiFactory[] = [
  createApiFactory({
    api: scmIntegrationsApiRef,
    deps: { configApi: configApiRef },
    factory: ({ configApi }) => ScmIntegrationsApi.fromConfig(configApi),
  }),
  ScmAuth.createDefaultApiFactory(),
  createApiFactory({
    api: keycloakAuthApiRef,
    deps: {
        discoveryApi: discoveryApiRef,
        oauthRequestApi: oauthRequestApiRef,
        configApi: configApiRef,
    },
    factory: ({ discoveryApi, oauthRequestApi, configApi }) =>
      OAuth2.create({
        discoveryApi,
        oauthRequestApi,
        provider: {
          id: 'keycloak',
          title: 'Keycloak Provider',
          icon: () => null,
        },
        defaultScopes: ["openid", "email", "profile", "offline_access"],
        environment: configApi.getOptionalString('auth.environment'),
      }),
    }),
];



## packages/app/src/App.tsx 
   
### Importa o módulo:

+import { apis, keycloakAuthApiRef } from './apis';

### Adicione a função:

const oauthProvider: SignInProviderConfig = {
  id: 'keycloak',
  title: 'Autenticação',
  message: 'Autenticar usando o Keycloak',
  apiRef: keycloakAuthApiRef,
};

const app = createApp({
  apis,
  components: {
    SignInPage: (props) => {
      return <SignInPage {...props} auto={false} provider={oauthProvider} />
    },
  },
  .
  
  .
  
  .
});


## Instala pacote JWT para decoder do token

yarn install jwt-decode

## Edite o arquivo packages/backend/src/plugins/auth.ts

Aqui esta o arquivo completo

import { stringifyEntityRef } from '@backstage/catalog-model';
import {
  createRouter,
  providers,
  defaultAuthProviderFactories,
} from '@backstage/plugin-auth-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';
import jwtDecoder from 'jwt-decode'


export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    logger: env.logger,
    config: env.config,
    database: env.database,
    discovery: env.discovery,
    tokenManager: env.tokenManager,
    providerFactories: {
      ...defaultAuthProviderFactories,

      // This replaces the default GitHub auth provider with a customized one.
      // The `signIn` option enables sign-in for this provider, using the
      // identity resolution logic that's provided in the `resolver` callback.
      //
      // This particular resolver makes all users share a single "guest" identity.
      // It should only be used for testing and trying out Backstage.
      //
      // If you want to use a production ready resolver you can switch to
      // the one that is commented out below, it looks up a user entity in the
      // catalog using the GitHub username of the authenticated user.
      // That resolver requires you to have user entities populated in the catalog,
      // for example using https://backstage.io/docs/integrations/github/org
      //
      // There are other resolvers to choose from, and you can also create
      // your own, see the auth documentation for more details:
      //
      //   https://backstage.io/docs/auth/identity-resolver
      github: providers.github.create({
        signIn: {
          resolver(_, ctx) {
            const userRef = 'user:default/guest'; // Must be a full entity reference
            return ctx.issueToken({
              claims: {
                sub: userRef, // The user's own identity
                ent: [userRef], // A list of identities that the user claims ownership through
              },
            });
          },
          // resolver: providers.github.resolvers.usernameMatchingUserEntityName(),
        },
      }),
      keycloak: providers.oauth2.create({
        signIn: {
          resolver({result}, ctx) {
            if (!result.params?.id_token) {
              throw new Error('Request did not contain a id_token') 
            }
            const decode = jwtDecoder<{preferred_username: string}>(result.params.id_token)
            
            const userRef = stringifyEntityRef({
              kind: 'User',
              name: decode.preferred_username,
              namespace: 'keycloak',
            });
            return ctx.issueToken({
              claims: {
                sub: userRef,
                ent: [userRef],
              },
            });
          },
        },
      }),
    },
  });
}

## Edite o arquivo app-config.yaml

app:
  title: Scaffolded Backstage App
  baseUrl: https://fqdn-backstage
  
.

.

.
backend:
  # Used for enabling authentication, secret is shared by all backend plugins
  # See https://backstage.io/docs/tutorials/backend-to-backend-auth for
  # information on the format
  # auth:
  #   keys:
  #     - secret: ${BACKEND_SECRET}
  baseUrl: https://fqdn-backstage
  listen:
    port: 7007
    
  .
  
  .
  
  .
  
cors:
    origin: https://fqdn-backstage
    methods: [GET, HEAD, PATCH, POST, PUT, DELETE]
    credentials: true
    
  .
 
  .
  
  .
  
 auth:
  environment: production
  providers:
    keycloak:
      production:
        clientId: "backstage"
        clientSecret: "HASH-CLIENT-SECRET"
        authorizationUrl: "https://fqdn.keycloak/auth/realms/RELM-Funcionarios/protocol/openid-connect/auth" ## Versão mais recente remova o "auth" do endpoint
        tokenUrl: "https://fqdn.keycloak/auth/realms/RELM-Funcionarios/protocol/openid-connect/token"  ## Versão mais recente remova o "auth" do endpoint
        
        
   

