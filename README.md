# halloumi

Pulumi + Heroku = Halloumi

You write your application, we run it in the cloud.

`halloumi` is a prototype and not under active development. 

## Summary

The halloumi SDK defines an application model where you simply write your http handler, and we take care of the rest. 

The [node](./nodejs) version is independently runnable via `yarn start`, and the [go](./go) version has a small binary CLI to orchestrate execution. 

All you have to do is define HTTP handlers, and `halloumi` takes care of running it in the cloud:

### Node.js

```typescript
// this application will deploy multiple services publicly available int the cloud
const application = new App();

const svc1 = new WebService("hello", (expressApp: express.Express) => {
    // a simple static handler
    expressApp.get('/', (req, res) => res.send("omg i'm alive\n"));
});
application.addService(svc1);


const svc2 = new WebService("world", (expressApp: express.Express) => {
    expressApp.get('/', async (req, res) => {
        // track dependencies across services and make them available at runtime
        const svc1Url = svc1.discover();
        // this handler calls svc1 and passes the result through
        const result = await (await fetch(svc1Url)).text()
        
        res.send(`this is the world. With a window from hello:\n${result} \n`);
    });
});
application.addService(svc2);

application.run().catch(err => { console.error(err) });
```

### Go

```go
app := app.NewApp("petStore")

// a cloud web service that returns a number [0-100)
randomNumberService := web.NewWebService("randomNumber", func() http.Handler {
    // a normal HTTP handler, halloumi takes care of running this in the cloud
    r := mux.NewRouter()
    handler := func(w http.ResponseWriter, r *http.Request) {
        rand.Seed(time.Now().UnixNano())
        fmt.Fprintf(w, "%d", rand.Intn(100))
    }
    r.HandleFunc("/", handler)
    return r
})

app.Register(randomNumberService)
```

### Cross-service Dependencies
Halloumi allows consuming services from other services:

```go
// a cloud web service that returns N of a random animal.
// this service takes a dependency on the URLs of the previously defined services
nAnimalsService := web.NewWebService("nAnimals", func() http.Handler {
    r := mux.NewRouter()
    var num string
    var animal string
    handler := func(w http.ResponseWriter, r *http.Request) {
        // Notice how we have the URL of randomNumberService
        // available to consume here!
        numResp, err := http.Get(randomNumberService.URL())
        if err != nil {
            fmt.Fprintf(w, err.Error())
        }
        defer numResp.Body.Close()

        if numResp.StatusCode == http.StatusOK {
            bodyBytes, err := ioutil.ReadAll(numResp.Body)
            if err != nil {
                log.Fatal(err)
            }
            num = string(bodyBytes)
        }

        // Notice how we have the URL of randomAnimalService
        // available to consume here!
        animalResp, err := http.Get(randomAnimalService.URL())
        if err != nil {
            fmt.Fprintf(w, err.Error())
        }
        defer numResp.Body.Close()

        if animalResp.StatusCode == http.StatusOK {
            bodyBytes, err := ioutil.ReadAll(animalResp.Body)
            if err != nil {
                log.Fatal(err)
            }
            animal = string(bodyBytes)
        }

        fmt.Fprintf(w, "Wow, you got %s %ss!", num, animal)
    }
    r.HandleFunc("/", handler)
    return r
})
```