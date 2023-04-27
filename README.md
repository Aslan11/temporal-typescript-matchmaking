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

Here's a simple example of a temporal workflow in TypeScript for matchmaking in the context of video games. This example assumes that you have a basic understanding of Temporal and have set up the necessary dependencies. We'll create a simple matchmaking system that will match players based on their skill level, location, and how long they've been waiting to be matched.

1. First, let's create a `Player` interface:
```typescript
interface Player {
  id: string;
  skillLevel: number;
  location: string;
  waitTime: number;
}
```
2. Next, we'll create a matchmaking activity:
```typescript
import { sleep } from '@temporalio/workflow';

export async function matchPlayers(players: Player[]): Promise<[Player, Player] | null> {
  const maxWaitTimeMs = 120000; // Set a maximum time to wait for a match, in this case 2 minutes
  const startTime = Date.now();

  while (Date.now() - startTime < maxWaitTimeMs) {
    // loop through the players
    for (let i = 0; i < players.length; i++) {
      for (let j = i + 1; j < players.length; j++) {
        // determine the skill difference 
        const skillDifference = Math.abs(players[i].skillLevel - players[j].skillLevel);
        // determine the location of the players (assuming location is a string representing a region)
        const sameLocation = players[i].location === players[j].location;
        // Determine if any player has been waiting longer than 30 seconds, 
        // this way we can priortize them if they haven't found a good match based on skill or location
        const anyPlayerWaitingLong = players[i].waitTime >= 30000 || players[j].waitTime >= 30000;

        // First try to match on skill level & location proximity,
        // otherwise use any player who has been waiting what we consider to be a long time, 30 seconds
        if ((skillDifference <= 10 && sameLocation) || anyPlayerWaitingLong) {
          // Match Found!
          const player1 = players[i];
          const player2 = players[j];

          // Remove players from the queue of players to be matched
          players.splice(j, 1);
          players.splice(i, 1);

          // Return them
          return [player1, player2];
        }
      }
    }

    // Wait for a while before checking again
    await sleep(1000);
  }

  // No match found within the given time
  return null;
}
```
3. Now, let's create a matchmaking workflow:
```typescript
import { defineSignal, setListener, sleep } from '@temporalio/workflow';
import { matchPlayers } from './matchmaking-activity';

export const addPlayerSignal = defineSignal<Player>('addPlayer');

export async function matchmakingWorkflow(): Promise<void> {
  const players: Player[] = [];

  setListener(addPlayerSignal, (player: Player) => {
    players.push({ ...player, waitTime: 0 });
  });

  while (true) {
    if (players.length >= 2) {
      const match = await matchPlayers(players);
      if (match) {
        console.log(`Match found: ${match[0].id} vs ${match[1].id}`);
      } else {
        // Increment the waitTime of all players in the queue
        for (let i = 0; i < players.length; i++) {
          players[i].waitTime += 1000;
        }
      }
    }
    await sleep(1000);
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
