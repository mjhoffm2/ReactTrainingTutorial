# Overview
In this tutorial series, I will be putting together an application similar to Slack.  This will serve as a guide on how to lay the groundwork for a production-ready application using the best practices.  This will also serve as an introduction to a new developer to technologies such as React, Redux, and TypeScript.

As a part of this tutorial series, I will also be implementing a back end using C#, .NET Core, and SQL Server.  However, the teaching elements of this tutorial will be mostly focused on the UI and will be as independent as possible from the server architecture used.

# Slack Clone

 ## Description

The Slack clone that we will be putting together in this application was chosen because it is something that almost any developer will be familiar with.  It is conceptually simple, but still provides enough depth to showcase many technologies and designs.  While Slack provides a ton of various features and utilities, we will limit the scope of our clone to just the channels and messaging portion, similar to an IRC service.

## Requirements

In our Slack clone, here is a high level overview of the actions that users should be able to perform:
 - Add Users
 - Edit Users
 - Soft-Delete Users
 - Create Channels
	 - Private Channels
	 - Public Channels
 - Edit Channels
	 - Rename channel
	 - Change channel owner
	 - Change who can invite to this channel
 - Soft-Delete Channels
 - Search for Channels
 - Join Channels
 - Invite Users to Channels
 - Leave Channels
 - Send Messages
	 - To a channel
	 - To a specific user (direct message)
 - Hard-Delete Messages

Additional requirements
 - There is automatically a 'General' public channel that all users are automatically a member of.
 - When a channel owner leaves, the user who has been a channel member the longest becomes the new owner.
 - A private channel with no members should be deleted.
 - Messages support special markup that gets rendered differently from plain text.

## Implementation notes

In this application, messaging will be done through websockets, but most other things will be handled through a REST API.  It is important to keep in mind that the concept of joining/leaving a channel is separate from connecting/disconnecting from it.  A member who has joined the channel but is not connected to it is simply offline, but a member who has not joined or has left the channel can not connect.

When connected to a channel, a socket is created and the following events are automatically broadcast:
 - A user joins, is edited, or leaves the channel
 - The channel is edited or deleted
 - A message is sent or deleted in the channel
 - A user connects or disconnects from the channel (comes online/goes offline)

The following entities are managed through a REST API and not through a socket.
 - Users
 - Channels (creating, searching, editing, etc)

## Entities

### User
 - UserId (Primary Key)
 - Username (Unique Key, cannot be changed, used for authentication)
 - Display Name
 - Email
 - Status
 - Description

### Channel
 - ChannelId (Primary Key)
 - Owner (Optional, Foreign Key)
 - Display Name
 - IsPublic (boolean)
 - CanAnyoneInvite (boolean)
 - IsGeneral (boolean)
 - IsActiveDirectMessage (boolean)

### Message
 - MessageId (Primary Key)
 - ChannelId (Foreign Key)
 - UserId (Foreign Key)
 - Timestamp
 - Message
 - IsEdited (boolean)

### UserChannel
 - UserChannelId (Primary Key)
 - UserId (Foreign Key, Unique Key part 1)
 - ChannelId (Foreign Key, Unique Key part 2)
 - JoinDate