# SRS-014 Fallback

## Fallback response logic

On the consumer side, it is desirable to use fallback response logic in those cases where the application business logic allows such a scenario. Typically, this refers to calculating a less accurate but acceptable service response when the provider is unavailable/errors are received from the provider.

<img width="919" alt="Screenshot 2024-12-11 at 16 41 37" src="https://github.com/user-attachments/assets/e79d749b-96c8-4710-9d89-051a1e1cbe41" />

In the example below, the payment service, upon receiving an error response from the fraud check service, switches to a scenario where fraud checking is assumed to be absent. In this case, the payment service consumer is returned a response without an error from the fraud check service.

<img width="1320" alt="Screenshot 2024-12-11 at 16 42 44" src="https://github.com/user-attachments/assets/a3a54500-c737-4b2b-88dd-a4a36d04da1c" />

Depending on the business scenario, fallback response logic may include returning values, for example, from cache, [graceful degradation](https://livebook.manning.com/book/microservices-in-action/chapter-6/131), when a less accurate but acceptable algorithm is applied to calculate the result, and static values.

<img src="https://blog.codecentric.de/_next/image?url=https%3A%2F%2Fmedia.graphassets.com%2Foutput%3Dformat%3Awebp%2FP1aoYAgMQ8aq5K77hIea&w=1920&q=75" />
