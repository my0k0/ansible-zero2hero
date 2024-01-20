## Configure DB
**Admin config**

```sh
#1) disable security (/etc/mongod.conf) --> Restart Mongodb 
#>  security:
#>    authorization: "disabled"

#2) drop admin user if exist
mongo admin --eval 'db.dropUser("superadmin")'
#3) create admin user
mongo admin --eval 'db.createUser({ user: "superadmin", pwd: "passwd123", roles: [{ role: "clusterAdmin", db: "admin" }, { role: "userAdminAnyDatabase", db: "admin" }] })'

#4) enable security (/etc/mongod.conf) --> Restart Mongodb
#>  security
#>    authorization: "enabled"
```

**App user config**
```sh
mongo admin --eval 'db.createUser({user: "todo", pwd: "todo", roles: [{ role: "readWrite, db: "test" }] })'
```

**Network config**
- Replace the value of bindIp in /etc/mongod.conf
```sh
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mongod.conf
```

## Run manuall
** Check connectivity with DB**
```sh
nc -zv <db-hostname> 27017
```

**From source code**
```sh
export DB_CONNECTION=mongodb://user:pass@localhost:27017 DB_NAME=test
go run main.go
```

**Or From build artifact**
```sh
export DB_CONNECTION=mongodb://user:pass@localhost:27017 DB_NAME=test
/artifact-build-file
```

## Run Production grade
1 - Create Systemd Service Todo which populates (2) & manages (3) -> notify: `systemctl reload-daemon`
2 - Put env vars in separated file (`/etc/todo.env`)
3 - Move build file from `/tmp/todo` to a standard place (`/usr/local/bin/todo`) -> notify: `systemctl restart todo`

## Deploy
- systemd service


## Run Production grade
```sh
npm install pm2 -g
# Set REACT_APP_API_ENDPOINT, then
# Start APP with pm2
pm2 start node_modules/react-scripts/scripts/start.js --name todo
```
