# Stage 1: Backend (Node.js)
# Use the official NGINX image as the base
FROM nginx:alpine

# Set the working directory
WORKDIR /usr/share/nginx/html

# Copy the HTML file into the NGINX directory
COPY index.html .

# Start NGINX when the container launches
CMD ["nginx", "-g", "daemon off;"]
