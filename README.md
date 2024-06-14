# Prisma/D1 Bug: IDs of optional relations are lost quietly during migration (prisma/prisma#24540)

## Setup
- `npm i`
- `npx wrangler d1 migrations apply prisma-demo-db --local`
- View the `.sqlite` file in `.wrangler/state/v3/minflare-D1DatabaseObject`. You can see that there is one row in the `User` and one row in the `Post` table. As expected, the `Post` has a `authorId` of `1`.
- Now uncomment the line in the `schema.prisma` file
- `npx prisma migrate diff --from-local-d1 --to-schema-datamodel ./prisma/schema.prisma --script --output prisma/migrations/0003_modify-table.sql`
- `npx wrangler d1 migrations apply prisma-demo-db --local`
- See how the `authorId` is now `NULL` :-(