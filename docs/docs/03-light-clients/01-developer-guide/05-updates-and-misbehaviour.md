---
title: Handling Updates and Misbehaviour
sidebar_label: Handling Updates and Misbehaviour
sidebar_position: 5
slug: /ibc/light-clients/updates-and-misbehaviour
---


# Handling `ClientMessage`s: updates and misbehaviour

As mentioned before in the documentation about [implementing the `ConsensusState` interface](04-consensus-state.md), [`ClientMessage`](https://github.com/cosmos/ibc-go/blob/v7.0.0/modules/core/exported/client.go#L147) is an interface used to update an IBC client. This update may be performed by:

- a single header,
- a batch of headers,
- evidence of misbehaviour,
- or any type which when verified produces a change to the consensus state of the IBC client.

This interface has been purposefully kept generic in order to give the maximum amount of flexibility to the light client implementer.

## Implementing the `ClientMessage` interface

Find the `ClientMessage` interface in `modules/core/exported`:

```go
type ClientMessage interface {
  proto.Message

  ClientType() string
  ValidateBasic() error
}
```

The `ClientMessage` will be passed to the client to be used in [`UpdateClient`](https://github.com/cosmos/ibc-go/blob/v7.0.0/modules/core/02-client/keeper/client.go#L48), which retrieves the `LightClientModule` by client type (parsed from the client ID available in `MsgUpdateClient`). This `LightClientModule` implements the [`LightClientModule` interface](02-light-client-module.md) for its specific consenus type (e.g. Tendermint).

`UpdateClient` will then handle a number of cases including misbehaviour and/or updating the consensus state, utilizing the specific methods defined in the relevant `LightClientModule`.

```go
VerifyClientMessage(ctx sdk.Context, clientID string, clientMsg ClientMessage) error
CheckForMisbehaviour(ctx sdk.Context, clientID string, clientMsg ClientMessage) bool
UpdateStateOnMisbehaviour(ctx sdk.Context, clientID string, clientMsg ClientMessage)
UpdateState(ctx sdk.Context, clientID string, clientMsg ClientMessage) []Height
```

## Handling updates and misbehaviour

The functions for handling updates to a light client and evidence of misbehaviour are all found in the [`LightClientModule`](https://github.com/cosmos/ibc-go/blob/501a8462345da099144efe91d495bfcfa18d760d/modules/core/exported/client.go#L51) interface, and will be discussed below.

> It is important to note that `Misbehaviour` in this particular context is referring to misbehaviour on the chain level intended to fool the light client. This will be defined by each light client.

## `VerifyClientMessage`

`VerifyClientMessage` must verify a `ClientMessage`. A `ClientMessage` could be a `Header`, `Misbehaviour`, or batch update. To understand how to implement a `ClientMessage`, please refer to the [Implementing the `ClientMessage` interface](#implementing-the-clientmessage-interface) section.

It must handle each type of `ClientMessage` appropriately. Calls to `CheckForMisbehaviour`, `UpdateState`, and `UpdateStateOnMisbehaviour` will assume that the content of the `ClientMessage` has been verified and can be trusted. An error should be returned if the `ClientMessage` fails to verify.

For an example of a `VerifyClientMessage` implementation, please check the [Tendermint light client](https://github.com/cosmos/ibc-go/blob/76730ff030b52a351096ee941b7e4da44af9f059/modules/light-clients/07-tendermint/update.go#L23).

## `CheckForMisbehaviour`

Checks for evidence of a misbehaviour in `Header` or `Misbehaviour` type. It assumes the `ClientMessage` has already been verified.

For an example of a `CheckForMisbehaviour` implementation, please check the [Tendermint light client](https://github.com/cosmos/ibc-go/blob/76730ff030b52a351096ee941b7e4da44af9f059/modules/light-clients/07-tendermint/misbehaviour_handle.go#L22).

> The Tendermint light client [defines `Misbehaviour`](https://github.com/cosmos/ibc-go/blob/v7.0.0/modules/light-clients/07-tendermint/misbehaviour.go) as two different types of situations: a situation where two conflicting `Header`s with the same height have been submitted to update a client's `ConsensusState` within the same trusting period, or that the two conflicting `Header`s have been submitted at different heights but the consensus states are not in the correct monotonic time ordering (BFT time violation). More explicitly, updating to a new height must have a timestamp greater than the previous consensus state, or, if inserting a consensus at a past height, then time must be less than those heights which come after and greater than heights which come before.

## `UpdateStateOnMisbehaviour`

`UpdateStateOnMisbehaviour` should perform appropriate state changes on a client state given that misbehaviour has been detected and verified. This method should only be called when misbehaviour is detected, as it does not perform any misbehaviour checks. Notably, it should freeze the client so that calling the `Status` function on the associated client state no longer returns `Active`.

For an example of a `UpdateStateOnMisbehaviour` implementation, please check the [Tendermint light client](https://github.com/cosmos/ibc-go/blob/76730ff030b52a351096ee941b7e4da44af9f059/modules/light-clients/07-tendermint/update.go#L202).

## `UpdateState`

`UpdateState` updates and stores as necessary any associated information for an IBC client, such as the `ClientState` and corresponding `ConsensusState`. It should perform a no-op on duplicate updates.

It assumes the `ClientMessage` has already been verified.

For an example of a `UpdateState` implementation, please check the [Tendermint light client](https://github.com/cosmos/ibc-go/blob/76730ff030b52a351096ee941b7e4da44af9f059/modules/light-clients/07-tendermint/update.go#L134).

## Putting it all together

The `02-client` `Keeper` module in ibc-go offers a reference as to how these functions will be used to [update the client](https://github.com/cosmos/ibc-go/blob/v7.0.0/modules/core/02-client/keeper/client.go#L48).

```go
clientModule, found := k.router.GetRoute(clientID)
if !found {
  return errorsmod.Wrap(types.ErrRouteNotFound, clientID)
}

if err := clientModule.VerifyClientMessage(ctx, clientID, clientMsg); err != nil {
  return err
}

foundMisbehaviour := clientModule.CheckForMisbehaviour(ctx, clientID, clientMsg)
if foundMisbehaviour {
  clientModule.UpdateStateOnMisbehaviour(ctx, clientID, clientMsg)
  // emit misbehaviour event
  return 
}

clientModule.UpdateState(ctx, clientID, clientMsg) // expects no-op on duplicate header
// emit update event
return
```
