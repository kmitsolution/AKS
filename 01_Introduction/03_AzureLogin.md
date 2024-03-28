# Azure Login using Azure CLI
To log in to Azure and set the subscription using Azure CLI, follow these steps:

1. **Install Azure CLI**: If you haven't already installed Azure CLI, you can download and install it from the official Azure CLI documentation: [Install Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

2. **Log in to Azure**: Open your command-line interface (CLI) and run the following command to log in to your Azure account:

   ```
   az login
   ```

   This command will open a browser window where you'll be prompted to sign in with your Azure account credentials. Once you've signed in, you can close the browser window and return to the CLI.

3. **List Subscriptions**: After logging in, you can list the Azure subscriptions associated with your account using the following command:

   ```
   az account list --output table
   ```

   This command will display a list of subscriptions along with their details such as subscription ID, subscription name, and tenant ID.

4. **Set Subscription**: Once you've identified the subscription you want to use, you can set it as the active subscription using its subscription ID or name. Use one of the following commands:

   Set subscription by subscription ID:
   ```
   az account set --subscription <subscription-id>
   ```

   Set subscription by subscription name:
   ```
   az account set --subscription "<subscription-name>"
   ```

   Replace `<subscription-id>` with the ID of the subscription you want to use, or `<subscription-name>` with the name of the subscription (enclosed in double quotes if it contains spaces).

5. **Verify Subscription**: You can verify that the correct subscription is set by running the following command:

   ```
   az account show
   ```

   This command will display detailed information about the currently selected subscription.

Once you've completed these steps, you'll be logged in to Azure CLI with the specified subscription set as the active subscription. You can now proceed to execute commands and manage Azure resources within the context of the selected subscription.
