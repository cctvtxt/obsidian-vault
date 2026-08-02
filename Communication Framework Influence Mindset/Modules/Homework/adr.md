страныне решения

- чтобы шарить common сделал в руте монорепозитория общую папку, не делал отдельный npm-пакет, мне кажется, что так почище и прозрачнее. Для мое задачи можно обойтись без npm 
- кажется с этим мы упремся в то что не сможем сделать изолированный докер образ
- решил сделать едлиный id для сущностей user и account чтобы таскать в jwt только его
- на инициативы 
- много где 1 аккаунт но много профилей для смены




общий id для аккаунта и профиля - отделил сущности чтобы можно было создавать множество профилей, но сейчас это лишнее
разделение просто по бизнес логике микросервисов


Не придумал, нужно ли разделять такие вещи на 2 отдельных метода
message GetSessionsRequest {
    optional string accountId = 1;
}
для пользователя и для админа чтобы в 1 случае id брался из метаданных с jwt

# в прод доабвить потом data.seed.ts

import { PrismaClient } from './generated' const prisma = new PrismaClient() const roles = [ { name: 'admin' }, { name: 'user' }, { name: 'moderator' }, ] async function main() { for (const role of roles) { await prisma.accountRole.upsert({ where: { name: role.name }, update: {}, // не перезаписывать если уже есть create: role, }) } } main() .catch(console.error) .finally(() => prisma.$disconnect())

# джобу

import { CronJob } from 'cron'; new CronJob('*/15 * * * *', async () => { const deleted = await prisma.session.deleteMany({ where: { expiresAt: { lt: new Date() } } }); console.log(`Deleted ${deleted.count} expired sessions`); }).start();

# sid по сессии
# aud по аккаунту и профилю



# добавлять везде где использую конф Модуль
        "start:dev": "cross-env NODE_ENV=development PORT=3001 tsx watch src/main.ts",
        "start:debug": "cross-env NODE_ENV=debug PORT=3001 tsx watch --inspect src/main.ts",
        "start:prod": "cross-env NODE_ENV=production PORT=3001 tsx watch src/main.ts",






feat(user-service): implement CreateUser method

feat(token-service): add mock GenerateTokens method

///

add fastify configuration

feat(api-gateway): configure global pipes, filters, interceptors

feat(common): add shared validation rules for user fields

1 chore(common): add configuration module

1.1 add api-gateway config

bootstrap api-gatway service

2 chore(api-gateway): add swagger module

3 feat(auth-service): add password hashing

4 feat(auth-service): add registration dto

5 feat(auth-service): implement register rpc

6 feat(auth-service): configure grpc server

8 feat(api-gateway): add registration route

9 feat(api-gateway): configure swagger

10 feat(user-service): create user on registration

7 feat(auth-service): add prisma migration file

7 feat(user-service): add prisma migration file

8 test(api-gateway): add integration registration tests

а как разделять бд в зависимости от того какое выбрано NODE ENV

//// add node-env

//

![[Pasted image 20260725162823.png]]

authentication \ jwt-based-auth - 7 hours (passport etc)

login, logout - add roles check in gateway

authorization

create-user - add authorization on gateway at this level

read-user

delete-user

update-user

..

не забыть grpc-exception-filter 5 hours

Проблемы по категориям


db-reset-auth: cd auth-service && node -e "require('child_process').execSync('npx prisma db execute --stdin',{input:'DROP SCHEMA public CASCADE; CREATE SCHEMA public;',stdio:['pipe','inherit','inherit']})"

db-reset-user: cd user-service && node -e "require('child_process').execSync('npx prisma db execute --stdin',{input:'DROP SCHEMA public CASCADE; CREATE SCHEMA public;',stdio:['pipe','inherit','inherit']})"

db-reset: db-reset-auth db-reset-user




# джобу чистить сессии