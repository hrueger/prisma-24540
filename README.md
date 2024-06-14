# Prisma/D1 Bug: IDs of optional relations are lost quietly during migration (prisma/prisma#24540)

## Setup
- `npm i`
- `npx wrangler d1 migrations apply prisma-demo-db --local`
- View the `.sqlite` file in `.wrangler/state/v3/minflare-D1DatabaseObject`. You can see that there is one row in the `User` and one row in the `Post` table. As expected, the `Post` has a `authorId` of `1`.
- Now uncomment the line in the `schema.prisma` file
- `npx prisma migrate diff --from-local-d1 --to-schema-datamodel ./prisma/schema.prisma --script --output prisma/migrations/0003_modify-table.sql`
- `npx wrangler d1 migrations apply prisma-demo-db --local`
- See how the `authorId` is now `NULL` :-(

## Possible fix
Copy the id pairs to temporary tables and then copy them back after the migration. Example migration file to test:
```sql
-- RedefineTables
PRAGMA defer_foreign_keys=ON;
PRAGMA foreign_keys=OFF;

-- Since D1 currently looses ids when the table is connected, copy the id / authorId pairs from Post to a temporary table (only if authorId is not null)
CREATE TABLE "__temp_post_authorId" (
    "id" TEXT NOT NULL,
    "authorId" INTEGER NOT NULL
);
INSERT INTO "__temp_post_authorId" ("id", "authorId") SELECT "id", "authorId" FROM "Post" WHERE "authorId" IS NOT NULL;

CREATE TABLE "new_User" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "email" TEXT NOT NULL,
    "name" TEXT NOT NULL DEFAULT 'Anonymous'
);
INSERT INTO "new_User" ("email", "id") SELECT "email", "id" FROM "User";
DROP TABLE "User";
ALTER TABLE "new_User" RENAME TO "User";
CREATE UNIQUE INDEX "User_email_key" ON "User"("email");

-- Copy the data from the temporary table back to Post
UPDATE "Post" SET "authorId" = (SELECT "authorId" FROM "__temp_post_authorId" WHERE "__temp_post_authorId"."id" = "Post"."id") WHERE "id" IN (SELECT "id" FROM "__temp_post_authorId");
DROP TABLE "__temp_post_authorId";

PRAGMA foreign_keys=ON;
PRAGMA defer_foreign_keys=OFF;
```