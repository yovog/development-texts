# Process of adding a new string using the Translation component in sm_platform
The following steps represent a semi-automated (yet human) process for translations using Github for collaboration, continuous integration and change tracking. When changes are made with a string that uses the Translate component or translate() function in a PR, they should ensure that the correspoding Chinese and Russian language files are also updated.
The steps to update are as follows:
* A developer working on a specific functionality that requires translation populates the new values in the `en.json` file by running the translations script `npm run translations`. The script includes a `formatjs` extract command that extracts messages from the source code and generates translation messages into the desired format in the `en.json` output file. The other `ru.json` and `zh.json` that are found together with the `en.json` file are added and updated manually, ensuring that the `IDs` in both match the `IDs` in the `en.json` file.
* The developer creates a pull request with the code changes including the updated `en.json` file. The pull request can be landed without being blocked by translations.
* The developer creates another translation pull request in the same repository with manually populated values in `ru.json` and `zh.json` in English. Both pull requests should be linked in their descriptions.
* In the description of the translation pull request the developer tags the translators and the QA to be notified via Github email that there is a translation request. In addition, the developer attaches `snippets` with the corresponding `IDs` and English `words` showing the platform changes to provide context for the translators.
* The developer creates a translation request thread in Teams and tags the translators and the QA to be notified further. The thread should include the translation `pull request link`, translation `deadline`, target `clients/projects` so that the translators and the QA are aware and can plan the changes and the verification.
* A ticket with the translation `pull request link` and the Teams `thread link` should be created to track the progress.
* The translators use the Teams thread to coordinate translating and post their suggested translations there.
* The developer should push the approved translations to the `ru.json` and `zh.json` files in the translation pull request, test them in their local development environment and merge them afterwards.
* The QA verifies the landed translations on the staging environment.