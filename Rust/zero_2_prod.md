[week of 2/06/23](/Rust/week_of_2_06_23.md)<br>
[week of 2/13/23](/Rust/week_of_2_13_23.md)<br>
[week of 2/20/23](/Rust/week_of_2_20_23.md)<br>
[week of 2/27/23](/Rust/week_of_2_27_23.md)<br> 
[week of 3/06/23](/Rust/week_of_3_06_23.md)<br> 
[week of 3/13/23](/Rust/week_of_3_13_23.md)<br> 
[week of 3/20/23](/Rust/week_of_3_20_23.md)<br> 
[week of 3/27/23](/Rust/week_of_3_27_23.md)<br> 
[week of 4/03/23](/Rust/week_of_4_03_23.md)<br> 
[week of 4/10/23](/Rust/week_of_4_10_23.md)<br> 
[week of 4/17/23](/Rust/week_of_4_17_23.md)<br> 
[week of 4/24/23](/Rust/week_of_4_24_23.md)<br> 

[previous week](/React/template_files.md)

to check all ports: lsof -i -P | grep -i "listen"
check specific port: sudo lsof -nP -iTCP:$PORT | grep LISTEN
kill port: sudo kill -9 <PID>

[lsof commands](https://phoenixnap.com/kb/lsof-command)

Digital Ocean

    doctl apps create --spec spec.yaml
    doctl auth init
    doctl auth init --access-token "Dig Ocean token" (https://github.com/digitalocean/doctl/issues/281)
    doctl apps list


Docker
    docker build --tag zero2prod --file Dockerfile .

    docker run zero2prod
    
Postgres instance
    ./scripts/init_db.sh

    sqlx migrate run

    SKIP_DOCKER=true ./scripts/init_db.sh

