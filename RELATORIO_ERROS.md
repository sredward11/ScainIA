# Relatório de Erros — Desenvolvimento do ScanIA

Este relatório registra os erros encontrados durante a estruturação do projeto, verificação de tipos, lint, diagnóstico de dependências e preparação para testes com **Expo Go** e **Supabase**.

---

## 1. Erros na Etapa de Estruturação e Tipagem (TypeScript / Expo SDK 58)

O guia de referência do projeto foi originalmente validado para o **Expo SDK 57 / React Native 0.86**. Ao inicializar o projeto com `create-expo-app@latest`, o ambiente instalado utilizou **Expo SDK 58 / React Native 0.88.1 / TypeScript 6.0.3**, o que gerou incompatibilidades reais detectadas na etapa de verificação:

### 1.1 Erro `TS2591: Cannot find name 'process'` em `src/lib/supabase.ts`
- **Onde ocorreu:** `src/lib/supabase.ts` (linhas 6 e 7), ao ler `process.env.EXPO_PUBLIC_SUPABASE_URL` e `process.env.EXPO_PUBLIC_SUPABASE_KEY`.
- **Causa:** O arquivo `expo-env.d.ts` (referenciado no `tsconfig.json`) ainda não havia sido gerado porque o servidor Metro (`npx expo start`) ainda não tinha sido executado.
- **Solução aplicada:** Criação do arquivo `expo-env.d.ts` com `/// <reference types="expo/types" />`, restaurando as definições globais de `process.env`.

### 1.2 Erro `TS2322` com `ColorValue` no `tabBarIcon` em `app/(app)/(tabs)/_layout.tsx`
- **Onde ocorreu:** `app/(app)/(tabs)/_layout.tsx` (linha 9).
- **Mensagem:** `Type '____ColorValue_Internal' is not assignable to type 'string | NativeColorValue | undefined'. Type 'null' is not assignable to type ...` e posteriormente `Type 'number' is not assignable to type ...`.
- **Causa:** No React Native 0.88, o tipo `ColorValue` passou a incluir `null` e `number`, enquanto o componente `<Ionicons>` de `@expo/vector-icons` aceita apenas `string | OpaqueColorValue | undefined`.
- **Solução aplicada:** Normalização explícita do valor recebido antes de repassar ao ícone:
  ```ts
  const iconColor = typeof color === 'number' || color === null ? undefined : color;
  ```

### 1.3 Erro `TS2769` no `RefreshControl` em `app/(app)/(tabs)/index.tsx`
- **Onde ocorreu:** `app/(app)/(tabs)/index.tsx` (linha 41).
- **Mensagem:** `Type '(options?: RefetchOptions | undefined) => Promise<QueryObserverResult<Receipt[], Error>>' is not assignable to type '() => void | Promise<void>'`.
- **Causa:** A tipagem do `RefreshControlProps.onRefresh` no React Native 0.88 + TypeScript 6 exige retorno estritamente `void | Promise<void>`, não aceitando o retorno `Promise<QueryObserverResult<...>>` da função `refetch` do TanStack Query v5.
- **Solução aplicada:** Envolvimento da chamada em `onRefresh={() => void refetch()}`.

---

## 2. Erros de Lint e Diagnóstico de Dependências (`expo lint` / `expo-doctor`)

### 2.1 Falha inicial do `npx expo lint` sem ESLint configurado
- **Onde ocorreu:** Primeira execução de `npx expo lint`.
- **Causa:** O template `blank-typescript` não inclui ESLint pré-instalado. O próprio comando instalou `eslint@^9.0.0` e `eslint-config-expo@~58.0.4`, mas falhou na mesma execução com `CommandError: npx expo lint requires ESLint version 8.57.0 or greater` antes de recarregar o binário recém-instalado.
- **Solução aplicada:** Nova execução de `npx expo lint` após a instalação automática concluir.

### 2.2 `Missing peer dependency: expo-font` no `npx expo-doctor`
- **Onde ocorreu:** Diagnóstico `npx expo-doctor` (19/20 verificações aprovadas, 1 falha).
- **Causa:** O pacote `@expo/vector-icons` exige `expo-font` como *peer dependency*, mas ele não constava na lista inicial de instalação da Etapa 2.2 do guia.
- **Solução aplicada:** Instalação via `npx expo install expo-font`, elevando o `expo-doctor` para **20/20 checks passed**.

### 2.3 Erro `react/display-name` em `app/(app)/(tabs)/_layout.tsx`
- **Onde ocorreu:** `app/(app)/(tabs)/_layout.tsx` (linha 8).
- **Causa:** A função auxiliar `tabIcon` retornava uma *arrow function* anônima interpretada como componente React sem nome pelo ESLint.
- **Solução aplicada:** Retorno de função nomeada `function TabIcon(...)`.

### 2.4 Erro `react-hooks/immutability` em `src/components/DonutChart.tsx`
- **Onde ocorreu:** `src/components/DonutChart.tsx` (linha 34).
- **Mensagem:** `Cannot reassign variable after render completes` na variável `consumed += length` dentro de `data.map(...)`.
- **Causa:** O plugin do React Compiler presente no `eslint-config-expo` SDK 58 proíbe mutação de variáveis externas durante o render.
- **Solução aplicada:** Substituição de `let consumed` + `.map()` por `.reduce()` puramente imutável para calcular `length` e `offset` de cada arco.

---

## 3. Pontos de Erro e Validação no Expo Go e Supabase (Testes de Integração)

Durante a integração e validação em dispositivo físico (Expo Go) e backend (Supabase), os seguintes cenários de erro foram previstos/tratados na arquitetura e devem ser validados no roteiro final:

| Área | Sintoma / Erro | Causa | Solução / Tratamento implementado |
| :--- | :--- | :--- | :--- |
| **Expo Go (Rede)** | QR Code não abre o aplicativo na rede local/universidade | Computador e celular em sub-redes diferentes ou firewall bloqueando portas do Metro | Iniciar com `npx expo start --tunnel` |
| **Expo Go (Cache/.env)** | Erro `"Variáveis do Supabase ausentes"` mesmo após editar `.env` | Cache do bundler Metro retendo variáveis antigas | Reiniciar limpando cache com `npx expo start -c` |
| **Expo Go (Permissões)** | Erro ao abrir câmera para fotografar nota fiscal | Permissão negada pelo usuário no sistema operacional | Tratamento em `pickReceiptImage` exibindo alerta orientando liberar a câmera nos ajustes do aparelho |
| **Supabase (Auth)** | Erro `"Confirme seu e-mail antes de entrar"` no cadastro de teste | Confirmação de e-mail ativada por padrão no Supabase Auth | Desativar *Confirm email* em `Authentication → Sign In / Providers → Email` para testes, ou confirmar o link enviado |
| **Supabase (RLS / Storage)** | `new row violates row-level security policy` ao enviar foto | Caminho do upload fora da pasta `<user_id>/...` ou políticas do `schema.sql` não aplicadas | O serviço `uploadReceiptImage` força o prefixo `${userId}/...` e `supabase/schema.sql` define as políticas para `receipts`, `receipt_items` e `storage.objects` |
| **Supabase (Edge Function)** | `Function not found (404)` ou erro de Docker no deploy | Função `scan-receipt` não publicada ou Docker ausente na máquina local | Publicar usando empacotamento remoto: `npx supabase functions deploy scan-receipt --use-api` |
| **Supabase (IA / Segredos)** | `"Falha ao processar a nota"` com status `500` ou `422` | Segredo `MOCK_AI` não definido como `true` ou `ANTHROPIC_API_KEY` ausente quando `MOCK_AI=false` | Configurar segredos via `npx supabase secrets set MOCK_AI=true` (modo simulado) ou definir `ANTHROPIC_API_KEY` |
