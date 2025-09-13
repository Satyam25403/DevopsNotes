Intro to CI/CD:
all devs push their code to a git repo. There is a server running serving the clients needs
CI/CD establishes the connection between ur actual single source of truth(i.e. the code base(git repo)) and the running production server

CI: taking up code, packaging it, installing dependencies, setting up the environment, making it ready to be shipped
CD: delivering or deploying the packaged, tested code into production or staging environments, making it live and usable for end users.


CI/CD—short for Continuous Integration and Continuous Delivery/Deployment—is essential in modern software development because it transforms how teams build, 
test, and release code. Here's why it's a game-changer:

🚀 Why CI/CD Is Needed
- Faster Releases
CI/CD automates the entire pipeline—from code commit to deployment—so teams can ship features and fixes in hours instead of weeks.
- Early Bug Detection
Automated tests run on every code change, catching errors before they hit production. This reduces costly late-stage debugging.
- Improved Collaboration
Developers, testers, and ops teams work in sync. Frequent commits to a shared branch prevent merge conflicts and siloed workflows.
- Consistent & Reliable Deployments
With automated builds and deployments, you eliminate manual steps that cause human error. Rollbacks are easier, and environments stay in sync.
- Customer-Centric Development
Frequent updates mean faster feedback loops. You can respond to user needs and market changes with agility.

🧪 Real-World Impact
Before CI/CD:
“Deployment was a long and risky process, taking days or sometimes weeks… bugs were discovered very late and were costly to fix.”

After CI/CD:
“Software delivery happens in hours instead of days… bugs are fixed quickly, rollbacks are easier.”

==================================================================================================================================================================

Some common types:
1. Self hosted: an intermediary server/software that has access to both git code and the server which is configured to pull the code from repo,
use ur own resources(everything self managed) and push to server to make the app live and running
an ex: Jenkins(self hosted CI/CD server)

2. VCS Embedded: does not require u to run ur own resources. for example ur code lives in git which gives u an inbuilt ci/cd which can access ur server.
VCS-version control system...u dont need to worry about storage, network or speed etc.
ex: github actions, gitlab workflows, bitbucket build, aws codebuild, gcp cloudbuild

3. External: very similar to vcs except that it is not the part of version control system architecture
u have a code base and a server and an intermediary that does the work of taking code and putting it on server but that is not self managed
it is fully managed by third party(configure ur git and server....everything else will be taken care by it)
ex: circle CI


