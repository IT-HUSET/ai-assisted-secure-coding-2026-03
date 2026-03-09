Take a look at the README.md to familiarize yourself with the project, the goal is to identify weakeness and vulnerabilities in the code, for both the application and its infrastructure.

# Mandates for application security

- Ensure commmon security headers are in place.
- Strict policy for cookies
- HSTS
- Traffic only via HTTPs
- Check the authorization model, everyone can consult a given file hash but only some should be able to submit, introduce a role for that if necessary.
- Make sure the containe runs the application as non-root.

# Mandates for infrastructure security

- All traffic within azure services should be internal (private endpoints/links).
- Vnet where the application and database reside, restrict traffic to be inside the vnet only.
- Include network security groups with default settings.
- Procure docker images that are light, and little to non critical vulnerabilities.
