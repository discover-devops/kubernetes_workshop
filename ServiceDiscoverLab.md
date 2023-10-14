To create a GitHub repository with the information you provided, you can follow these steps:

1. **Create a New Repository on GitHub:**
   - Go to https://github.com and log in to your GitHub account.
   - Click the "+" icon in the top right corner and select "New repository."
   - Choose a repository name, for example, "k8s-nginx-labs."
   - Choose the visibility (public or private) as per your preference.
   - You can add a README file, but we will provide the YAML files separately, so it's not necessary.

2. **Clone the Repository:**
   - Clone the newly created repository to your local machine using the following command, replacing `<repository_url>` with your repository's URL:
     ```bash
     git clone <repository_url>
     cd k8s-nginx-labs
     ```

3. **Create a Directory Structure:**
   - Create directories for each lab (Lab 1 and Lab 2) to keep your YAML files organized.

   ```bash
   mkdir Lab1 Lab2
   ```

4. **Create YAML Files:**
   - Create YAML files for each lab based on the information you provided. Use a text editor or code editor to create these files. You can use the `vi` or `nano` text editor on your terminal, or you can use a code editor like Visual Studio Code.

   - In your `k8s-nginx-labs` directory, you can create the YAML files as follows:

   **Lab 1 - Create a NodePort Service with Nginx Containers:**
   - Create a file named `my-nginx-deployment.yaml` inside the `Lab1` directory and paste the content for Lab 1 into this file.

   **Lab 2 - Creating a ClusterIP Service with Nginx Containers:**
   - Create a file named `nginx-deployment.yaml` inside the `Lab2` directory and paste the content for Lab 2 into this file.

   - Create a file named `nginx-service-clusterip.yaml` inside the `Lab2` directory and paste the content for creating a ClusterIP Service.

5. **Commit and Push Your Changes:**
   - After creating and saving the YAML files in their respective directories, commit the changes to your local repository and then push them to GitHub.

   ```bash
   git add .
   git commit -m "Add Lab 1 and Lab 2 YAML files"
   git push
   ```

6. **Verify on GitHub:**
   - Go to your GitHub repository in your web browser and verify that the YAML files have been successfully pushed.

Now you have a GitHub repository with the two lab directories, each containing the necessary YAML files for Lab 1 and Lab 2. You can access and share these files with others through your GitHub repository.
