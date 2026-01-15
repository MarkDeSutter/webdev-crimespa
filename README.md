# St. Paul Crime Single Page Application

A Vue.js-based single page application that visualizes crime data in St. Paul using geospatial mapping and interactive data visualization.

## Overview

This application provides insights into crime patterns across St. Paul's districts through an interactive web interface. It integrates geographic data with crime statistics to help users explore and understand crime distribution across different neighborhoods.

## Features

- Interactive geographic visualization of St. Paul districts
- Crime data exploration and filtering
- Single Page Application (SPA) built with Vue 3
- Responsive web design with Foundation CSS
- Real-time data updates

## Project Structure

```
├── index.html          # Main HTML entry point
├── package.json        # Project dependencies and scripts
├── vite.config.js      # Vite configuration
├── src/
│   ├── App.vue        # Root Vue component
│   └── main.js        # Application entry point
├── public/
│   ├── css/           # Stylesheets (Foundation, custom styles)
│   ├── js/            # JavaScript libraries (jQuery, Foundation)
│   └── data/          # GeoJSON data (St. Paul districts)
└── README.md          # This file
```

## Tech Stack

- **Frontend Framework**: Vue.js 3
- **Build Tool**: Vite
- **Map Library**: Leaflet (for interactive mapping)
- **CSS Framework**: Foundation
- **Data Format**: GeoJSON for geographic data
- **Dependencies**: jQuery for DOM manipulation

## Prerequisites

- Node.js (version 20.19.0 or later, or 22.12.0+)
- **webdev-rest** API server running (required for backend crime data)

## Installation

1. Clone the repository
2. Install dependencies:
   ```bash
   npm install
   ```

## Development

Make sure the **webdev-rest API server** is running before starting the application.

To run the development server on port 8080:

```bash
npm run dev -- --port 8080
```

The application will be available at `http://localhost:8080`.

Once the website has started, enter `http://locahost:8000` into the URL textbox.

## Authors

- Mark DeSutter (desu5994@stthomas.edu)
- Jair Garces Vargas
- Ibrahima Ka

## License

University of St. Thomas - CISC 375 Project

## Repository

[GitHub Repository](https://github.com/MarkDeSutter/webdev-crimespa)
