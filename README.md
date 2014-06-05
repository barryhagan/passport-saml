Passport-SAML
=============

This is a [SAML 2.0](http://en.wikipedia.org/wiki/SAML_2.0) authentication provider for [Passport](http://passportjs.org/), the Node.js authentication library.

The code was originally based on Michael Bosworth's [express-saml](https://github.com/bozzltron/express-saml) library.

Passport-SAML has been tested to work with both [SimpleSAMLphp](http://simplesamlphp.org/) based Identity Providers, and with [Active Directory Federation Services](http://en.wikipedia.org/wiki/Active_Directory_Federation_Services).

## Installation

    $ npm install passport-saml

## Usage

### Configure strategy

This example utilizes the [Feide OpenIdp identity provider](https://openidp.feide.no/). You need an account there to log in with this. You also need to [register your site](https://openidp.feide.no/simplesaml/module.php/metaedit/index.php) as a service provider.

The SAML identity provider will redirect you to the URL provided by the `path` configuration.

```javascript
passport.use(new SamlStrategy(
  {
    path: '/login/callback',
    entryPoint: 'https://openidp.feide.no/simplesaml/saml2/idp/SSOService.php',
    issuer: 'passport-saml'
  },
  function(profile, done) {
    findByEmail(profile.email, function(err, user) {
      if (err) {
        return done(err);
      }
      return done(null, user);
    });
  })
));
```

Config parameter details:
* `path`: path to callback; will be combined with protocol and server host information to construct callback url if `callbackUrl` is not specified (default: `/saml/consume`)
* `protocol`: protocol for callback; will be combined with path and server host information to construct callback url if `callbackUrl` is not specified (default: `https://`)
* `callbackUrl`: full callbackUrl (overrides path if supplied)
* `entryPoint`: identity provider entrypoint
* `issuer`: issuer string to supply to identity provider
* `cert`: see 'security and signatures'
* `privateCert`: see 'security and signatures'
* `decryptionPvk`: optional private key that will be used to attempt to decrypt any encrypted assertions that are received
* `identifierFormat`: if truthy, name identifier format to request from identity provider (default: `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`)

### Provide the authentication callback

You need to provide a route corresponding to the `path` configuration parameter given to the strategy:

```javascript
app.post('/login/callback',
  passport.authenticate('saml', { failureRedirect: '/', failureFlash: true }),
  function(req, res) {
    res.redirect('/');
  }
);
```

### Authenticate requests

Use `passport.authenticate()`, specifying `saml` as the strategy:

```javascript
app.get('/login',
  passport.authenticate('saml', { failureRedirect: '/', failureFlash: true }),
  function(req, res) {
    res.redirect('/');
  }
);
```

Additional config values supported:
* `samlFallback`: if set to `getAuthorizeUrl`, will initiate a redirect to identity provider on authentication failure

### generateServiceProviderMetadata( decryptionCert )

As a convenience, the strategy object exposes a `generateServiceProviderMetadata` method which will generate a service provider metadata document suitable for supplying to an identity provider.  This method will only work on strategies which are configured with a `callbackUrl` (since the relative path for the callback is not sufficient information to generate a complete metadata document).

The `decryptionCert` argument should be a certificate matching the `decryptionPvk` and is required if the strategy is configured with a `decryptionPvk`.


## Security and signatures

Passport-SAML uses the HTTP Redirect Binding for its `AuthnRequest`s, and expects to receive the messages back via the HTTP POST binding.

Authentication requests sent by Passport-SAML can be signed using RSA-SHA1. To sign them you need to provide a private key in the PEM format via the `privateCert` configuration key. For example:

```javascript
    privateCert: fs.readFileSync('./cert.pem', 'utf-8')
```

It is a good idea to validate the incoming SAML Responses. For this, you can provide the Identity Provider's certificate using the `cert` confguration key:

```javascript
    cert: 'MIICizCCAfQCCQCY8tKaMc0BMjANBgkqh ... W=='
```

## Usage with Active Directory Federation Services

Here is a configuration that has been proven to work with ADFS:

```javascript
  {
    entryPoint: 'https://ad.example.net/adfs/ls/',
    issuer: 'https://your-app.example.net/login/callback',
    callbackUrl: 'https://your-app.example.net/login/callback',
    cert: 'MIICizCCAfQCCQCY8tKaMc0BMjANBgkqh ... W==',
    identifierFormat: null
  }
```

Please note that ADFS needs to have a trust established to your service in order for this to work.
