# loopback4-ratelimiter

[![LoopBack](<https://github.com/strongloop/loopback-next/raw/master/docs/site/imgs/branding/Powered-by-LoopBack-Badge-(blue)-@2x.png>)](http://loopback.io/)

A simple loopback-next extension for rate limiting in loopback applications. This extension uses [express-rate-limit](https://github.com/nfriedly/express-rate-limit) under the hood with redis used as store for rate limiting key storage using [rate-limit-redis](https://github.com/wyattjoh/rate-limit-redis)

## Install

```sh
npm install loopback4-ratelimiter
```

## Usage

In order to use this component into your LoopBack application, please follow below steps.

- Add component to application.

```ts
this.component(RateLimiterComponent);
```

- Minimum configuration required for this component to work is the redis datasource name. Please provide it as below.

```ts
this.bind(RateLimitSecurityBindings.CONFIG).to({
  name: 'redis',
});
```

- By default, ratelimiter will be initialized with default options as mentioned [here](https://github.com/nfriedly/express-rate-limit#configuration-options). However, you can override any of the options using the Config Binding like below.

```ts
const rateLimitKeyGen = (req: Request) => {
  const token =
    (req.headers &&
      req.headers.authorization &&
      req.headers.authorization.replace(/bearer /i, '')) ||
    '';
  return token;
};

......

this.bind(RateLimitSecurityBindings.CONFIG).to({
  name: 'redis',
  max: 60,
  keyGenerator: rateLimitKeyGen,
});
```

- The component exposes a sequence action which can be added to your server sequence class. Adding this will trigger ratelimiter middleware for all the requests passing through.

```ts
export class MySequence implements SequenceHandler {
  constructor(
    @inject(SequenceActions.FIND_ROUTE) protected findRoute: FindRoute,
    @inject(SequenceActions.PARSE_PARAMS) protected parseParams: ParseParams,
    @inject(SequenceActions.INVOKE_METHOD) protected invoke: InvokeMethod,
    @inject(SequenceActions.SEND) public send: Send,
    @inject(SequenceActions.REJECT) public reject: Reject,
    @inject(RateLimitSecurityBindings.RATELIMIT_SECURITY_ACTION)
    protected rateLimitAction: RateLimitAction,
  ) {}

  async handle(context: RequestContext) {
    const requestTime = Date.now();
    try {
      const {request, response} = context;
      const route = this.findRoute(request);
      const args = await this.parseParams(request, route);

      // rate limit Action here
      await this.rateLimitAction(request, response);

      const result = await this.invoke(route, args);
      this.send(response, result);
    } catch (err) {
      ...
    } finally {
      ...
    }
  }
}
```

- This component also exposes a method decorator for cases where you want tp specify different rate limiting options at API method level. For example, you want to keep hard rate limit for unauthorized API requests and want to keep it softer for other API requests. In this case, the global config will be overwritten by the method decoration. Refer below.

```ts
const rateLimitKeyGen = (req: Request) => {
  const token =
    (req.headers &&
      req.headers.authorization &&
      req.headers.authorization.replace(/bearer /i, '')) ||
    '';
  return token;
};

.....

@ratelimit(true, {
  max: 60,
  keyGenerator: rateLimitKeyGen,
})
@patch(`/auth/change-password`, {
  responses: {
    [STATUS_CODE.OK]: {
      description: 'If User password successfully changed.',
    },
    ...ErrorCodes,
  },
  security: [
    {
      [STRATEGY.BEARER]: [],
    },
  ],
})
async resetPassword(
  @requestBody({
    content: {
      [CONTENT_TYPE.JSON]: {
        schema: getModelSchemaRef(ResetPassword, {partial: true}),
      },
    },
  })
  req: ResetPassword,
  @param.header.string('Authorization') auth: string,
): Promise<User> {
  return this.authService.changepassword(req, auth);
}
```

- You can also disable rate limiting for specific API methods using the decorator like below.

```ts
@ratelimit(false)
@authenticate(STRATEGY.BEARER)
@authorize(['*'])
@get('/auth/me', {
  description: ' To get the user details',
  security: [
    {
      [STRATEGY.BEARER]: [],
    },
  ],
  responses: {
    [STATUS_CODE.OK]: {
      description: 'User Object',
      content: {
        [CONTENT_TYPE.JSON]: AuthUser,
      },
    },
    ...ErrorCodes,
  },
})
async userDetails(
  @inject(RestBindings.Http.REQUEST) req: Request,
): Promise<AuthUser> {
  return this.authService.getme(req.headers.authorization);
}
```

## License

[MIT](https://github.com/ruturajmore/loopback4-ratelimiter/blob/master/LICENSE)
