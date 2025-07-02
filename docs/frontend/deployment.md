# 🛠️ Desarrollo del Frontend

## Configuración del Entorno

### Prerrequisitos

```bash
# Node.js y pnpm
node --version  # >= 18.0.0
pnpm --version  # >= 8.0.0 (recomendado)

# O usar npm/yarn
npm --version   # >= 9.0.0
yarn --version  # >= 1.22.0
```

### Instalación

```bash
# Clonar repositorio
git clone https://github.com/FFigueroa17/secure-dash-project.git
cd secure-dash-project/secure-dash-project

# Instalar dependencias
pnpm install
# o
npm install
```

### Variables de Entorno

```bash
# Copiar archivo de ejemplo
cp .env.example .env.local
```

```env
# .env.local
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000/ws
```

## 🚀 Comandos de Desarrollo

### Scripts Disponibles

```bash
# Servidor de desarrollo
pnpm dev
# Accesible en http://localhost:3000

# Build de producción
pnpm build

# Iniciar servidor de producción
pnpm start

# Linting y formateo
pnpm lint
pnpm lint:fix

# Type checking
pnpm type-check

# Tests
pnpm test
pnpm test:watch
```

### Estructura de Desarrollo

```
src/
├── app/
│   ├── (auth)/
│   │   └── page.tsx              # Página de login/registro
│   ├── (dashboard)/
│   │   ├── dashboard/
│   │   │   └── page.tsx          # Dashboard principal
│   │   ├── jails/
│   │   │   └── page.tsx          # Gestión de jails
│   │   └── logs/
│   │       └── page.tsx          # Viewer de logs
│   ├── api/
│   │   └── auth/                 # NextAuth endpoints
│   ├── layout.tsx                # Root layout
│   └── globals.css               # Estilos globales
├── components/
│   ├── ui/                       # shadcn/ui components
│   ├── forms/
│   │   ├── login-form.tsx
│   │   └── register-form.tsx
│   └── dashboard/
│       ├── metrics-card.tsx
│       ├── jails-table.tsx
│       └── logs-viewer.tsx
├── lib/
│   ├── auth.ts                   # NextAuth configuración
│   ├── utils.ts                  # Utilidades generales
│   └── validations.ts            # Esquemas Zod
└── types/
    ├── auth.ts                   # Tipos de autenticación
    └── api.ts                    # Tipos de API
```

## 🔧 Configuración Técnica

### Next.js Configuration

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    serverActions: true,
    serverComponentsExternalPackages: ['bcryptjs']
  },
  images: {
    domains: ['randomuser.me', 'images.unsplash.com']
  },
  env: {
    NEXTAUTH_URL: process.env.NEXTAUTH_URL,
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL
  }
}

module.exports = nextConfig
```

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  darkMode: 'class',
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
}
```

## 🎨 Desarrollo de Componentes

### Form Components con Zod

```tsx
// components/forms/login-form.tsx
'use client'

import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'
import { z } from 'zod'

const loginSchema = z.object({
  emailOrUsername: z.string().min(1, 'Required'),
  password: z.string().min(8, 'Minimum 8 characters'),
  rememberMe: z.boolean().optional()
})

type LoginFormData = z.infer<typeof loginSchema>

export function LoginForm({ onSubmit }: { onSubmit: (data: LoginFormData) => void }) {
  const form = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema)
  })

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        {/* Form fields */}
      </form>
    </Form>
  )
}
```

### API Integration

```tsx
// lib/api-client.ts
class ApiClient {
  private baseUrl: string

  constructor() {
    this.baseUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000'
  }

  async getJails() {
    const response = await fetch(`${this.baseUrl}/api/jails`)
    if (!response.ok) throw new Error('Failed to fetch jails')
    return response.json()
  }

  async getLogs(params?: { limit?: number; jail?: string }) {
    const searchParams = new URLSearchParams(params as Record<string, string>)
    const response = await fetch(`${this.baseUrl}/api/logs?${searchParams}`)
    return response.json()
  }
}

export const apiClient = new ApiClient()
```

### Server Actions

```tsx
// app/actions/auth.ts
'use server'

import { signIn, signOut } from '@/lib/auth'
import { loginSchema, registerSchema } from '@/lib/validations'

export async function signin(formData: FormData) {
  const rawData = {
    emailOrUsername: formData.get('emailOrUsername'),
    password: formData.get('password'),
    rememberMe: formData.get('rememberMe') === 'true'
  }

  const validatedData = loginSchema.parse(rawData)
  
  const result = await signIn('credentials', {
    ...validatedData,
    redirect: false
  })

  return result
}
```

## 📱 Responsive Design

### Breakpoints Tailwind

```css
/* Breakpoints utilizados */
sm: 640px   /* Tablet pequeña */
md: 768px   /* Tablet */
lg: 1024px  /* Desktop pequeño */
xl: 1280px  /* Desktop */
2xl: 1536px /* Desktop grande */
```

### Layout Responsivo

```tsx
// Layout adaptativo
<div className="min-h-screen flex flex-col lg:flex-row">
  {/* Auth form - mobile first */}
  <section className="flex-1 flex items-center justify-center p-4 sm:p-8">
    <div className="w-full max-w-md">
      {/* Form content */}
    </div>
  </section>
  
  {/* Hero section - hidden on mobile */}
  <section className="hidden lg:block flex-1 relative p-4">
    {/* Hero content */}
  </section>
</div>
```

!!! tip "Desarrollo Local"
    - Usa `pnpm dev` para hot reload automático
    - Configura VS Code con extensiones de Next.js y Tailwind
    - Usa TypeScript strict mode para mejor type safety

!!! warning "Consideraciones"
    - Siempre valida props con Zod schemas
    - Implementa error boundaries para componentes críticos
    - Usa Suspense para loading states

!!! success "Herramientas Recomendadas"
    - **VS Code** con extensiones Next.js y Tailwind
    - **React DevTools** para debugging
    - **Tailwind CSS IntelliSense** para autocompletado