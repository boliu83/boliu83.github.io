```javascript
var request = require('request')
  , getopts = require('getopts')

const options = getopts(process.argv.slice(2))

if ( options['ssl-verify'] ) {
    process.env["NODE_TLS_REJECT_UNAUTHORIZED"] = 1
} else {
    process.env["NODE_TLS_REJECT_UNAUTHORIZED"] = 0
}

var host = ""
  , password = ""

if ( options['host'] ) {
    host = options['host']
} else {
    process.stderr.write("ERROR: --host is missing\n")
    process.exit(-1)
}

if ( options['password'] ) {
    password = options['password']
} else {
    process.stderr.write("ERROR: --password is missing\n")
    process.exit(-2)
}

const j = request.jar()

var loginData = {
    uri : "https://" + host + "/cgi-bin/cgi",
    jar : j,
    form : {
        CSRF_TOKEN : 0,
        user : 'admin',
        password : password,
        lang : 0,
        login : 'Log in'
    },
    headers : [
        { name : 'content-type', value: 'application/x-www-form-urlencoded' }
    ]
}

request.post( loginData, (err, res, body) => {
    if ( ! err && res.statusCode != 200 ) {

        eventsData = {
            url : "https://" + host + "/cgi-bin/cgi?form=31",
            jar : j
        }

        request.get(eventsData, (err, res, body) => {

            let rows = body.match(/\<tr\>.*\<\/tr\>/g)
            let events = []
            if ( rows && rows.length ) {
                rows.forEach( row => {
                    let data = row.match(/\<td headers=...\>(.*?)\<\/td\>/g)

                    if ( data && data.length && data.length == 6 ) {
                        events.push({
                            logID : stripeTag(data[1]),
                            time : stripeTag(data[2]),
                            timeStamp: new Date(stripeTag(data[2])).getTime() / 1000,
                            failingSubsystem : stripeTag(data[3]),
                            severity: stripeTag(data[4]),
                            src : stripeTag(data[5]).trimEnd()
                        })
                    }

                })

                process.stdout.write(JSON.stringify({
                    updatedTS : parseInt(new Date().getTime() / 1000),
                    updatedAt : new Date(),
                    events : events
                }, " ", 4))

            } else {
                process.stdout.write(JSON.stringify({
                    updatedTS : parseInt(new Date().getTime() / 1000),
                    updatedAt : new Date(),
                    events : events
                }, " ", 4))
            }

            let matches = body.match(/CSRF_TOKEN\' value=\'([0-9]*)\'/)

            if ( matches && matches.length && matches.length > 1) {
                token = matches[1]
                logout(token)
            }

        })
    } else {
        // if login fails
        process.stdout.write("ERROR: ASM log in failed\n")
        process.stdout.write("BODY: " + body + "\n")
        process.exit(10)
    }
})

function stripeTag(text) {
    return text.replace(/<\/??td.*?>/g,'')
}

function logout(csrf_token) {
    var logout = {
        uri : "https://" + host + "/cgi-bin/cgi",
        jar : j,
        headers : [
            { name : 'content-type', value: 'application/x-www-form-urlencoded' }
        ],
        form : {
            CSRF_TOKEN : csrf_token,
            submit : "Log out"
        }
    }

    request.post(logout)
}
```

