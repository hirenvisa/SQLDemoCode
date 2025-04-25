```bash
 docker version
```

```bash
mkdir ~/postgres-volume
                    
docker run --name postgres \
    -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=password \
    -p 5432:5432 \
    -v ~/postgres-volume/:/var/lib/postgresql/data \
    -d postgres:latest

 docker container ls -f name=postgres
```


Validate container is up and running with below commands
```bash
 docker logs postgres
```

Connect to psql 
```bash
 docker exec -it postgres psql -U 
 
 \conninfo
```
You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432".













