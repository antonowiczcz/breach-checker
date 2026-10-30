## test 1 - docker run --rm --network breach-net healthcheck-probe
backend: OK
## test 2 - docker run --rm healthcheck-probe
backend: UNREACHABLE
## test 3
docker stop backend
docker run --rm --network breach-net healthcheck-probe
- backend: UNREACHABLE
docker start backend


## Dlaczego bez --network nazwa backend nie działa?

Kontener uruchomiony bez flagi --network trafia do domyślnej sieci typu bridge. W domyślnej sieci Dockera wbudowany serwer DNS nie działa – kontenery nie potrafią tłumaczyć nazw innych kontenerów na adresy IP. Aby translacja nazwy backend na IP była możliwa, oba kontenery muszą znajdować się w tej samej sieci użytkownika (user-defined bridge), w tym przypadku breach-net.

