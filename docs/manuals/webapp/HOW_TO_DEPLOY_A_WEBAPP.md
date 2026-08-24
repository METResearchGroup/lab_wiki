# How to deploy a webapp

For making studies, we need to deploy an app. We have been wanting to find a systematized way to do this. The typical way that a coding agent will tell you to do it is to use AWS. Do not do this. This is an overly complicated way of doing it that will take you longer than it needs to actually create it, plus it costs more.

What we can do instead is use a combination of the following:

- (Frontend) [Vercel](https://vercel.com/): Vercel hosts the part of the app that people open in a browser. You connect a GitHub repo, and Vercel builds the site, puts it on a public URL, and updates that URL when you push new code.
- (Backend) [Railway](https://railway.com/): Railway hosts the part of the app that runs in the background. You deploy a server, a database, or a worker, and Railway keeps that process running and gives you logs, env vars, and a URL for the API.

Use Railway for the backend, and use Vercel for the pages people see.

## Options

### Option 1: Full frontend and backend

For complicated apps with a fully-fledged frontend and backend, you want both Vercel and Railway.

### Option 2: Backend-only (API + static templates)

For simple apps, an API that serves static templates might be enough. In this case, you only need Railway.

### Which one is right for me?

Pass this prompt to your AI agent:

```markdown
Review my application and my specs. Advise on whether we need a full frontend and backend or just the backend only. We prefer the backend only if the application that we're building can be done easily enough with a simple, fast API plus static templates, and we prefer that for the majority of applications. On the side of creating a full Next.js app and a Railway backend, only if our experimental design is sufficiently complicated that we will need these steps.

Grill the user with the grill-me skill in https://github.com/mark-torres10/ai_tools/blob/main/skills/grill-me/SKILL.md to get more details, and then advise the user on a feasible next path.
```

## Setup

Create personal accounts for each of these providers.

Then set up the Vercel and Railway MCP tools. An MCP is a way for your coding agent, or however it is that you do your coding work, to connect to your accounts in these two providers. You can connect to them through the command line or through ChatGPT.

- [Vercel MCP setup link](https://vercel.com/docs/agent-resources/vercel-mcp)
- [Railway MCP setup link](https://docs.railway.com/ai/mcp-server)

A typical app setup for us will have the following:

- Vercel for the frontend
- Railway for the backend
- S3 to store data/assets

## Instructions to tell your AI Agent

Tell your AI agent the following, which will manage setting up the starter code for the frontend and backend, deploying it, and then prompt you to actually start implementation.

This assumes that you're using a tool like Cursor or Codex or Claude Code to do your work. Feel free to ask me (Mark) for alternative steps if not.

### Instructions for Option 1

If going with Option 1, give the full prompt.

```markdown
Create two folders, frontend/ and backend/.

- In frontend/, this will be a Next.js app, deployed to Vercel. Set up the Next.js app scaffolding. Create a "Hello World" example.
- In backend/, this will be a FastAPI bcakend, deployed to Railway. Set up the FastAPI app scaffolding. Create a "Hello World" example.

Then, make sure and prompt the user to set up the Vercel and Railway CLI and MCP access, if not already enabled. Check the environment for this.

Then check to see if Vercel has been linked to the current GitHub repo. Link it if not.

Once this is done, open a PR for the user with the Hello World frontend and backend examples. Then merge it in. Then review the Vercel and Railway deployments to see if they have been successfully deployed. Loop until you see successful deployments for both.

Then once done, give the user step-by-step instructions to connect the Railway backend to the Vercel deployed frontend. Use your CLI and MCP access to be as exact and prescriptive as possible, giving them the exact URLs and paths.

Loop until this is done and the user has connected the backend to the frontend.

Once done, provide the user the link to their frontend.

Then ask the user what needs to be built next, and walk them through this workflow:

- Get specs
- Build
- Open PR
- Merge PR
- See deployment

Create a docs/runbooks/ in the user's GitHub repo, if it doesn't exist, and add a HOW_TO_BUILD_FEATURES.md with that 5-step process.

Then update the user's AGENTS.md file (if it exists in root, else create it) to indicate (1) that we use Vercel and Railway for deployment, (2) code lives in frontend/ and backend/ and (3) we use the `HOW_TO_BUILD_FEATURES.md` to define how we ship code.
```

### Instructions for Option 2

If going with Option 2, give the following smaller prompt:

```markdown
Create a backend/ folder:

- In backend/, this will be a FastAPI bcakend, deployed to Railway. Set up the FastAPI app scaffolding. Create a "Hello World" example.

Then, make sure and prompt the user to set up Railway CLI and MCP access, if not already enabled. Check the environment for this.

Once this is done, open a PR for the user with the Hello World backend example. Then merge it in. Then review the Railway deployment to see if they have been successfully deployed. Loop until you see successful deployments. Then give the user the public-facing URL for the Railway server.

Then ask the user what needs to be built next, and walk them through this workflow:

- Get specs
- Build
- Open PR
- Merge PR
- See deployment

Create a docs/runbooks/ in the user's GitHub repo, if it doesn't exist, and add a HOW_TO_BUILD_FEATURES.md with that 5-step process.

Then update the user's AGENTS.md file (if it exists in root, else create it) to indicate (1) that we use Railway for deployment, (2) code lives in backend/ and (3) we use the `HOW_TO_BUILD_FEATURES.md` to define how we ship code.
```
