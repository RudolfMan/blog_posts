%{
  title: "Beware of potential N+1 in LiveView with PubSub",
  description: "Phoenix LiveView is awesome! Accompanied with Phoenix PubSub it provides the superpower for building interactive real-time UX. But "with great power comes great responsibility""
}
---
Lately I've been playing more with [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) and [Surface-UI](https://surface-ui.org). While I enjoy building rich UI with almost no custom JS and getting more used to thinking differently about interactivity, where some the state is pushed to the user from the Back-End not upon the request from the client, but on state update, I noticed a potential place where some sort of a N+1 problem, that I haven't dealt with before, could appear if developer is not careful (as with other types of N+1 problems).

During the development of the UI, sometimes I just open 2 browser windows on the same view to test broadcasted message pushed down to  client that didn't interact with UI but should receive an update.

It's important to keep in mind that LiveView spawns an erlang process for each connected client.
