# Use the official Odoo 17 image as a base
ARG ODOO_VERSION=17.0
FROM odoo:${ODOO_VERSION}

# Set environment variables for easier configuration (can be overridden in docker-compose)
ENV USER=odoo
ENV HOME=/home/odoo

# Install system dependencies if needed (e.g., for Python packages or specific addons)
# Example:
# USER root
# RUN apt-get update && apt-get install -y --no-install-recommends \
#     wkhtmltopdf \
#     python3-pip \
#     # Add other dependencies here
#     && rm -rf /var/lib/apt/lists/*
# USER ${USER}

# Install Python dependencies if needed (e.g., for custom addons)
# COPY --chown=${USER}:${USER} requirements.txt /home/odoo/
# RUN pip3 install --user -r /home/odoo/requirements.txt

# Expose Odoo ports
EXPOSE 8069 8071 8072

# The odoo.conf will be mounted via docker-compose.yml
# Custom addons will also be mounted via docker-compose.yml

# Set the default command (can be overridden)
# CMD ["odoo"] 