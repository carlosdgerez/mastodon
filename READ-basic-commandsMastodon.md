
## Basic Account Commands

| Action | Command | Description |
|------|------|------|
| Create account | `docker compose exec web bin/tootctl accounts create USERNAME --email EMAIL --confirmed` | Creates a new user and prints a generated password |
| Approve account | `docker compose exec web bin/tootctl accounts approve USERNAME` | Approves an account if registrations require approval |
| Promote to admin | `docker compose exec web bin/tootctl accounts modify USERNAME --role Owner` | Gives full administrator permissions |
| Reset password | `docker compose exec web bin/tootctl accounts modify USERNAME --reset-password` | Generates a new password for the user |
| Delete account | `docker compose exec web bin/tootctl accounts delete USERNAME` | Removes the user and all associated posts |

## Example

Create user:  
 ~~~Bash 
docker compose exec web bin/tootctl accounts create carlo --email carlo@localhost --confirmed
 ~~~




Approve account:
 ~~~Bash 
docker compose exec web bin/tootctl accounts approve carlo
 ~~~



Promote to administrator:

 ~~~Bash 
docker compose exec web bin/tootctl accounts modify carlo --role Owner
 ~~~
