

### UUID 생성
#pgcrypto  #gen_random_uuid

`id UUID NOT NULL DEFAULT gen_random_uuid()`
```SQL 
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE website
(
    id UUID PRIMARY KEY NOT NULL DEFAULT gen_random_uuid(),
    name VARCHAR(20) NOT NULL
);
```


