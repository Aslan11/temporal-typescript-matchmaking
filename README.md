# Matchmaking With Temporal & TypeScript!

Temporal is a workflow management system that simplifies the development of complex, stateful, and long-running applications, like matchmaking systems in video games. Here are some benefits of using Temporal for matchmaking:

- **Fault tolerance**: Temporal provides built-in fault tolerance by preserving the state of your workflows, which allows the matchmaking process to continue even after system crashes, worker failures, or network issues.
- **Scalability**: Temporal can manage a large number of concurrent matchmaking workflows and scale horizontally as the number of players and match requests increases. This helps you maintain a robust matchmaking system that can handle the growing demands of your game.
- **Simplified development**: Temporal allows you to express complex matchmaking logic as simple, modular code. By separating the matchmaking logic into activities and workflows, you can easily develop, test, and maintain your matchmaking system.
- **Time-based operations**: Temporal provides first-class support for time-based operations, like timeouts, delays, and schedules which are essential for matchmaking systems. You can manage player waiting times, timeouts for match searches, and other time-sensitive aspects of your matchmaking system with ease.
- **External event handling**: Temporal supports handling external events, such as player disconnections or new player arrivals, during the matchmaking process. You can use signals to integrate these events into your matchmaking workflows, ensuring that your system remains responsive to real-time changes in the player pool.
- **Long-running workflows**: Matchmaking systems may need to manage long-running workflows, especially when dealing with large player pools and varying player availability. Temporal is designed to handle long-running workflows and maintain their state over extended periods, ensuring that your matchmaking system remains accurate and reliable.
- **Visibility and monitoring**: Temporal provides tools and dashboards for monitoring and tracking the status of your matchmaking workflows. This allows you to gain insights into the performance of your matchmaking system, identify potential bottlenecks, and make improvements as needed.

Overall, Temporal offers a powerful and flexible platform for building matchmaking systems in video games, enabling you to create a reliable, scalable, and fault-tolerant system that can handle the complex and dynamic requirements of a real-world gaming environment.

Now, let's dive in!

## Example

Here's a simple example of a Temporal workflow in TypeScript for matchmaking in the context of video games. This example assumes that you have a basic understanding of Temporal and have set up the necessary dependencies. We'll create a simple matchmaking system that will match players based on their skill level, location, and how long they've been waiting to be matched.

Disclaimer:  The current approach with a single workflow handling all incoming signals can become a bottleneck, especially when dealing with a high volume of incoming player signals. To address this issue, you can use multiple workflow instances with sharding, for the sake of simplicity I decided not to include that in this example.

1. First, let's create a `Player` interface:
```typescript
interface Player {
  id: string;
  skillLevel: number;
  location: string;
  startedWaiting: number;
}
```
2. Next, we'll create a matchmaking activity:
```typescript
export async function matchPlayers(players: Player[]): Promise<[Player, Player] | null> {
  for (let i = 0; i < players.length; i++) {
    for (let j = i + 1; j < players.length; j++) {
      // calculate difference in skill
      const skillDifference = Math.abs(players[i].skillLevel - players[j].skillLevel);
      // determine whether the players are in the same region
      const sameLocation = players[i].location === players[j].location;
      // determine how long the players have been waiting
      const currentTime = Date.now();
      const LongTime = 30000
      const player1WaitTime = currentTime - players[i].startedWaiting;
      const player2WaitTime = currentTime - players[j].startedWaiting;
      const player1WaitingLong = player1WaitTime >= longTime
      const player2WaitingLong = player2WaitTime >= longTime
      
      // if any player has been waiting a long time, prioritize that over skill & location
      if ((skillDifference <= 10 && sameLocation) || (player1WaitingLong && player2WaitingLong)) {
        // return the matched players
        return [players[i], players[j];
      }
    }
  }

  // No match found
  return null;
}

```
3. Now, let's create a matchmaking workflow:
```typescript
import {Context, ContinueAsNew, defineSignal, proxyActivities, setListener } from '@temporalio/workflow';
import type * as activities from './activities';
const { matchPlayers } = proxyActivities<typeof activities>({
  startToCloseTimeout: '1 minute',
});

export interface MatchmakingWorkflowArgs {
  players: Player[];
}

export const addPlayerSignal = defineSignal<Player>('addPlayer');

async function* newPlayerGenerator(players: Player[]): AsyncGenerator<Player> {
  let index = 0;
  while (true) {
    if (index < players.length) {
      yield players[index];
      index++;
    } else {
      await new Promise<void>((resolve) => {
        setListener(addPlayerSignal, (player: Player) => {
          players.push({ ...player, startedWaiting: Date.now() });
          resolve();
        });
      });
    }
  }
}

export async function matchmakingWorkflow(args: MatchmakingWorkflowArgs): Promise<void> {
  const { players } = args;
  const startTime = Date.now();
  const continueAsNewIntervalMs = 3600000; // Restart workflow every hour

  for await (const player of newPlayerGenerator(players)) {
    if (Date.now() - startTime > continueAsNewIntervalMs) {
      // Continue as new with the current state
      const continueAsNew: ContinueAsNew = Context.configure({ taskQueue: Context.info().taskQueue });
      continueAsNew.run(matchmakingWorkflow, { players });
      return;
    }

    if (players.length >= 2) {
      const match = await matchPlayers(players);
      if (match) {
        console.log(`Match found: ${match[0].id} vs ${match[1].id}`);

        // Remove matched players from the original players array in the workflow
        players.splice(players.indexOf(match[0]), 1);
        players.splice(players.indexOf(match[1]), 1);
      }
    }
  }
}
```
4. Finally, let's create a simple worker to run the workflow:
```typescript
import { Worker } from '@temporalio/worker';

async function run() {
  const worker = await Worker.create({
    workflowsPath: require.resolve('./matchmaking-workflow'),
    activitiesPath: require.resolve('./matchmaking-activity'),
    taskQueue: 'matchmaking',
  });

  await worker.run();
}

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```
