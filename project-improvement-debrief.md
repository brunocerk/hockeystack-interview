## Short Debrief

For this project, I would discuss the adoption of Typescript with the team as it helps directly with code readability and consistency.
Allowing faster development of features and bug fixes by enforcing correct ways to interact with the internal and third-party code,
a better team work by enforcing market and team wise best practices, code rules and preferences, and helping on knowledge
sharing and team onboarding.

Creating an integrations module with a Hubspot integration abstraction in it, would prevent code
repetition and maintain consistency across the codebase while improving integrations reliability, readability, maintainability, 
error handling, and this together with Typescript is very powerful.

Implementing a unified logging module using tools like Winston with structured logging would improve log visibility and
provide developers more tools for a faster bug/behaviour identification and fix, also helping the integration of log
tracing tools for visualization and smart storage.

Regarding the project architecture, pushing more and more the usage of the event-based architecture with microservices 
would enable better performance and consistency of events processing. Performance-wise, parallel data 
synchronization should be utilized wherever synchronizations are independent, also strategies like multi-thread 
synchronization, fallback/dead-letter queue and idempotency should also help with consistency and reliability of the system. 
Smart load-balancing based on business, analytics and error states data, should not only help with performance but 
cost management and predictability as effective load-balancing techniques would help control processing rates, preventing
slowdowns/failures across the system and unpredictable costs due to variable event throughput.


<details>
  <summary> Longer Debrief</summary>


## Project Architecture

### Webhooks

Hubspot provides a webhook api for listening to events like create/update of entities like contacts and companies ( https://developers.hubspot.com/beta-docs/guides/api/app-management/webhooks#webhook-subscriptions )
but it unfortunately does not work the same way for Engagements like Meetings ( https://community.hubspot.com/t5/APIs-Integrations/Web-hook-trigger-for-Engagement-events-like-Meeting-etc/td-p/313903 )

### Push usage of event-based architecture and microservices

Consumer microservices do a better job of consuming events ( specially from webhooks ) and producing
events to the system for reactivity ( triggering other services like notifications, data synchronizations, integration actions... ) due to the possibility of parallel processing and fallback and retry strategies.
And with a good load-balancing the system can better control the processing rate to prevent slowdown/stoppage and unpredictable costs due to the constant changing of throughput of events.

### Parallel data synchronization

A very common scalability and reliability problem with data synchronization is related to serially synchronizing different entities at the same process/thread and workflow.
```text
Sync Contacts > Sync Companies > Sync Meetings > Done > Repeat
```
So whenever possible, that means, when one sync does not depend on another. Parallel synchronization should be used.
```text
Sync Contacts / Sync Companies / Sync Meetings > All Done > Repeat
```

This also enhances the chance of one sync failing/slowing other syncs
and often times high priority workflows are negatively impacted by low priority ones
```text
Sync Contacts / Sync Companies / Sync Meetings > All Done > Repeat
```

A good strategies are:
- Single synchronization microservice that decides what entity to sync at creation
- Multiple synchronization microservice, one for each entity
- Multi thread synchronization if technologically allowed
- Smart load-balancing that can horizontally and vertically balance sync microservices using data like business/account priority, time since last sync, errors happening, third-party failures...
  - ex: Hubspot API is failing with a known error for server issue: lower the instances of hubspot related sync services
  - ex: High priority account is at peak usage hours: increases instances and processing power of sync related services

## Code Improvements

### Code consistency

One of the important points for me was to maintain code consistency as in a real scenario, unless explicitly required and agreed, it's not a good
behaviour to change working critical features because of a new one. But it's clear that many points of the code could be improved and that would require a study, team discussions and a planned effort.

### Usage of Typescript

Typescript could be used to improve many aspects of the code

- Code readability: Quicker and better interaction with the code, allowing features and bugs to be developed faster
- Code reliability: Better visualization of correct usage of internal functions and integrations interactions ( like hubspot/api-client )
- Code consistency: Typescript has better tools for enforcing best practices and avoiding bad ones throughout configurations and even code patterns.

### Unified module for Hubspot API interactions

A good practice is to create an Integration module with abstractions for the many interactions with integrations like Hubspot.
Some of the improvements this could provide:

- Prevent code repetition for common api interactions and custom logics like refreshAccessToken, retry/pagination/timeout strategy...
```javascript
    while (tryCount <= 4) {
    try {
        searchResult = await hubspotClient.crm.companies.searchApi.doSearch(searchObject);
        break;
    } catch (err) {
        tryCount++;

        if (new Date() > expirationDate) await refreshAccessToken(domain, hubId);

        await new Promise((resolve, reject) => setTimeout(resolve, 5000 * Math.pow(2, tryCount)));
    }
}
```

- Helps maintain code usage consistency across code base, also, together with Typescript, makes this even more powerful.

- Better error handling of code interactions and also api/networking issues specific to integrations

  - Hubspot has custom error codes for different scenarios, having at least the most important ones mapped is a good practice
    https://developers.hubspot.com/beta-docs/reference/api/other-resources/error-handling

  - ```javascript
    if (!searchResult) throw new Error('Failed to fetch companies for the 4th time. Aborting.');
    ```


### Logging and Error Handling

- Unified logging strategy through the creation of a logging module to be reused across the code
```javascript
const { createLogger, format, transports } = require('winston');

const logger = createLogger({
  level: 'info',
  exitOnError: false,
  format: format.json(),
  transports: [
    new transports.File({ filename: `${appRoot}/logs/<FILE_NAME>.log` }),
  ],
});

module.exports = logger;

// worker-service.js
const workerLogger = logger.child({service: 'worker-service'})
workerLogger.info('Hello log with metas', {color: 'blue' });
```

- Use of structured logging for better log visibility and faster bug/behaviour identification.
```javascript
workerLogger.error('Something went wrong', {importantId: entity.id, importantTimestamp: entity.createdAt });
```

- Mapping of common errors like database connection, third-party error codes, internal workflows and how critical/retryable they are.
This can be used for retry workflows and smart failing out of code workflows.

</details>