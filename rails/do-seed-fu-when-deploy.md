# デプロイ時にseed_fuを走るようにする

- Capfile

```
require 'seed-fu/capistrano3'
```

- config/deploy.rb

```
before 'deploy:publishing', 'db:seed_fu'
```
