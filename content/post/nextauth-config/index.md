---
title: NextAuth配置教程
description: 使用 NextAuth.js 实现认证功能的完整配置指南
date: 2024-10-20
categories:
    - frontend-world
tags:
    - next.js
    - authentication
    - prisma
image: cover.png
---


# NextAuth配置教程

>  尝试使用Prisma作为数据库适配器,并配置GitHub OAuth和邮箱密码登录。

## 简述next-auth配置

next-auth是用来给next项目增加认证的，既可以配置邮箱也可以配置GitHub，Google等授权认证，整个使用流程相对简单，而且似乎配置GitHub认证好像其实数据库都可以不要。暂时还处于摸索阶段，勉勉强强配置好了登录，实际上当前配置的逻辑还有点问题，例如GitHub授权后user表中有邮箱，这时候如果邮箱注册会冲突，不过处理起来也还好，暂时先不做了。

关于使用，客户端中提供useSession拿到用户信息，服务端中通过getServerSession获取用户信息。总的来说配置和使用上手难度还是比较容易的。但是配置Google的时候总是报错，测试可能是服务器连不上，但是我开了全局也不行，后面再试试。
![image](https://github.com/user-attachments/assets/57ee1d11-065c-4f85-a8b1-e9046f76767c)

下面用配置好的代码，用Claude生成了大致的步骤

## 步骤1: 安装依赖

首先,安装必要的依赖:

```bash
npm install next-auth @prisma/client @next-auth/prisma-adapter bcryptjs
npm install prisma --save-dev
```

## 步骤2: 配置环境变量

在`.env`文件中添加以下环境变量:

```env
DATABASE_URL="your_database_url_here"
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your_nextauth_secret_here
GITHUB_ID=your_github_client_id
GITHUB_SECRET=your_github_client_secret
```

## 步骤3: 设置Prisma

初始化Prisma并创建数据库schema:

```bash
npx prisma init
```

在`prisma/schema.prisma`文件中定义您的模型。这里是一个基本的用户模型示例:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Account {
  id                 String  @id @default(cuid())
  userId             String
  type               String
  provider           String
  providerAccountId  String
  refresh_token      String?  @db.Text
  access_token       String?  @db.Text
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String?  @db.Text
  session_state      String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  password      String?
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

```

然后,运行以下命令来创建数据库表:

```bash
npx prisma db push 
```

## 步骤4: 创建NextAuth配置文件

创建`lib/auth.ts`文件并添加以下内容:

```typescript
import {NextAuthOptions} from "next-auth";
import {PrismaAdapter} from "@next-auth/prisma-adapter";
import {PrismaClient} from "@prisma/client";
import CredentialsProvider from "next-auth/providers/credentials";
import GitHubProvider from "next-auth/providers/github";
import bcrypt from "bcryptjs";

// 创建 Prisma 客户端实例
const prisma = new PrismaClient();

// 定义 NextAuth 的配置选项
export const authOptions: NextAuthOptions = {
    // 使用 Prisma 适配器来处理数据库操作
    adapter: PrismaAdapter(prisma),

    // 配置身份验证提供者
    providers: [
        // 使用邮箱密码的凭证提供者
        CredentialsProvider({
            name: "Credentials",
            credentials: {
                email: {label: "Email", type: "text"},
                password: {label: "Password", type: "password"}
            },
            // 验证用户凭证
            async authorize(credentials) {
                if (!credentials?.email || !credentials?.password) {
                    throw new Error("Missing credentials");
                }
                // 在数据库中查找用户
                const user = await prisma.user.findUnique({
                    where: {email: credentials.email}
                });
                if (!user || !user.password) {
                    throw new Error("User not found");
                }
                // 验证密码
                const isPasswordValid = await bcrypt.compare(credentials.password, user.password);
                if (!isPasswordValid) {
                    throw new Error("Invalid password");
                }
                return user;
            }
        }),
        // GitHub OAuth 提供者
        GitHubProvider({
            clientId: process.env.GITHUB_ID!,
            clientSecret: process.env.GITHUB_SECRET!,
        }),
    ],

    // 自定义认证页面路径
    pages: {
        signIn: '/auth/signin',
        signOut: '/auth/signout',
        error: '/auth/error',
        verifyRequest: '/auth/verify-request',
    },

    // 使用 JWT 策略管理会话
    session: {
        strategy: "jwt",
    },

    // 回调函数配置
    callbacks: {
        // 登录回调
        async signIn({user, account, profile, email, credentials}) {
            try {
                console.log("Sign In Callback:", {user, account, profile, email});

                // 处理 GitHub 登录
                if (account && account.provider === 'github' && profile) {
                    // 查找现有用户
                    const existingUser = await prisma.user.findFirst({
                        where: {
                            OR: [
                                {email: profile.email},
                                {accounts: {some: {providerAccountId: account.providerAccountId}}}
                            ]
                        },
                        include: {accounts: true}
                    });

                    if (existingUser) {
                        // 如果用户存在但没有 GitHub 账号，则添加 GitHub 账号
                        if (!existingUser.accounts.some(acc => acc.provider === 'github')) {
                            await prisma.account.create({
                                data: {
                                    userId: existingUser.id,
                                    type: account.type,
                                    provider: account.provider,
                                    providerAccountId: account.providerAccountId,
                                    access_token: account.access_token,
                                    token_type: account.token_type,
                                    scope: account.scope,
                                }
                            });
                        }
                        return true;
                    }

                    // 如果用户不存在，创建新用户
                    await prisma.user.create({
                        data: {
                            name: profile.name || (profile as any).login || 'Unknown',
                            email: profile.email || `${account.providerAccountId}@github.com`,
                            accounts: {
                                create: {
                                    type: account.type,
                                    provider: account.provider,
                                    providerAccountId: account.providerAccountId,
                                    access_token: account.access_token,
                                    token_type: account.token_type,
                                    scope: account.scope,
                                }
                            }
                        }
                    });
                }

                return true;
            } catch (error) {
                console.error("Error in signIn callback:", error);
                return false;
            }
        },
        // JWT 回调
        async jwt({token, user, account}) {
            if (user) {
                token.id = user.id;
            }
            if (account && account.access_token) {
                token.accessToken = account.access_token;
            }
            return token;
        },
        // 会话回调
        async session({session, token}) {
            if (session.user) {
                (session.user as any).id = token.id as string;
                (session as any).accessToken = token.accessToken;
            }
            return session;
        },
        // 重定向回调
        async redirect({url, baseUrl}) {
            // 允许相对回调 URL
            if (url.startsWith("/")) return `${baseUrl}${url}`
            // 允许同源的回调 URL
            else if (new URL(url).origin === baseUrl) return url
            return baseUrl
        },
    },

    // 认证事件处理
    events: {
        async signIn(message) {
            console.log("User signed in:", message)
        },
        async signOut(message) {
            console.log("User signed out:", message)
        },
        async createUser(message) {
            console.log("User created:", message)
        },
        async linkAccount(message) {
            console.log("Account linked:", message)
        },
        async session(message) {
            console.log("Session created:", message)
        },
    },

    // 开发环境启用调试模式
    debug: process.env.NODE_ENV === 'development',

    // 日志配置
    logger: {
        error(code, metadata) {
            console.error(code, metadata);
        },
        warn(code) {
            console.warn(code);
        },
        debug(code, metadata) {
            console.log(code, metadata);
        },
    },
};

```

## 步骤5: 创建API路由

创建`app/api/auth/[...nextauth]/route.ts`文件:

```typescript
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);

export { handler as GET, handler as POST };
```

## 步骤6: 创建注册API

创建`app/api/auth/register/route.ts`文件:

```typescript
import { NextResponse } from 'next/server';
import bcrypt from 'bcryptjs';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function POST(req: Request) {
    try {
        const { name, email, password } = await req.json();

        const existingUser = await prisma.user.findUnique({ where: { email } });
        if (existingUser) {
            return NextResponse.json({ error: "User already exists" }, { status: 400 });
        }

        const hashedPassword = await bcrypt.hash(password, 10);
        const user = await prisma.user.create({
            data: { name, email, password: hashedPassword },
        });

        return NextResponse.json({ message: "User created successfully" }, { status: 201 });
    } catch (error) {
        console.error("Registration error:", error);
        return NextResponse.json({ error: "An error occurred during registration" }, { status: 500 });
    }
}
```

## 步骤7: 创建登录和注册页面

创建`app/auth/signin/page.tsx`和`app/auth/register/page.tsx`文件,实现登录和注册表单。

## 步骤8: 在应用中使用NextAuth

在您的组件中,您可以使用`useSession`钩子来获取当前会话状态:

```typescript
import { useSession } from "next-auth/react";

export default function Component() {
  const { data: session } = useSession();

  if (session) {
    return <p>Welcome, {session.user.name}!</p>;
  }

  return <p>Please sign in</p>;
}
```

## 步骤9: 保护路由

您可以使用`getServerSession`函数来保护服务器端路由:

```typescript
import { getServerSession } from "next-auth/next";
import { authOptions } from "@/lib/auth";

export default async function ProtectedPage() {
  const session = await getServerSession(authOptions);

  if (!session) {
    return <p>Access Denied</p>;
  }

  return <p>Welcome to the protected page, {session.user.name}!</p>;
}
```

这就是在Next.js项目中设置和配置NextAuth的基本步骤。根据您的具体需求,您可能需要进行更多自定义配置。
