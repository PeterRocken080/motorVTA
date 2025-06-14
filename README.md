# Motor  Vehicle Theft Analysis

**Spatial Analysis of Motor Vehicle Theft Incidents in Detroit-Michigan between 2018 to 2023**

The goal of this study used Point Pattern analysis to discover motor vehicle theft incidences in Detroit between the years 2018 and 2023, as well as to acquire insights into the spatial distribution, clustering, and underlying spatial patterns of motor theft occurrences. Such as a city or neighborhood. This research seeks to discover hotspots of motor theft incidents and understand their temporal trends over the years.

library(shiny)
library(leaflet)
library(sf)
library(dplyr)
library(lubridate)
library(ggplot2)
library(readr)

# Load and preprocess data
crime_data <- read_csv("Book1.csv")

# Parse date-time
crime_data <- crime_data %>%
  mutate(
    ibr_date = ymd_hms(ibr_date),
    date = as.Date(ibr_date),
    hour = hour(ibr_date),
    day_of_week = wday(ibr_date, label = TRUE),
    time_label = format(ibr_date, "%H:%M")
  )

# Convert to sf object
crime_sf <- st_as_sf(crime_data, coords = c("longitude", "latitude"), crs = 4326, remove = FALSE)

# Load shapefile of Detroit boundary
detroit_boundary <- st_read("City_of_Detroit_Boundary.shp")
detroit_boundary <- st_transform(detroit_boundary, crs = st_crs(crime_sf))

# Filter points within city limits
crime_sf <- st_join(crime_sf, detroit_boundary, join = st_within)

# UI
ui <- fluidPage(
  titlePanel("Spatial Analysis of Motor Vehicle Theft - Detroit 2020"),
  
  sidebarLayout(
    sidebarPanel(
      checkboxGroupInput("day_filter", "Select Day(s) of Week:",
                         choices = levels(crime_sf$day_of_week), selected = levels(crime_sf$day_of_week)),
      sliderInput("hour_filter", "Select Hour Range:",
                  min = 0, max = 23, value = c(0, 23)),
      selectInput("neighborhood_filter", "Select Neighborhood:",
                  choices = c("All", unique(crime_sf$neighborhood)), selected = "All")
    ),
    
    mainPanel(
      leafletOutput("theftMap", height = 500),
      br(),
      h4("Summary Statistics"),
      verbatimTextOutput("summaryStats"),
      br(),
      plotOutput("hourPlot"),
      plotOutput("dayPlot")
    )
  )
)

# Server
server <- function(input, output, session) {
  
  # Reactive filtered data
  filtered_data <- reactive({
    df <- crime_sf %>%
      filter(day_of_week %in% input$day_filter,
             hour >= input$hour_filter[1], hour <= input$hour_filter[2])
    if (input$neighborhood_filter != "All") {
      df <- df %>% filter(neighborhood == input$neighborhood_filter)
    }
    df
  })
  
  # Leaflet map
  output$theftMap <- renderLeaflet({
    leaflet() %>%
      addProviderTiles(providers$CartoDB.Positron) %>%
      addPolygons(data = detroit_boundary, color = "#444444", weight = 1, fill = FALSE) %>%
      addCircleMarkers(data = filtered_data(),
                       lng = ~longitude, lat = ~latitude,
                       popup = ~paste(address, "<br>", format(ibr_date, "%Y-%m-%d %H:%M")),
                       radius = 4, stroke = FALSE, fillOpacity = 0.6)
  })
  
  # Summary statistics
  output$summaryStats <- renderPrint({
    df <- filtered_data()
    total <- nrow(df)
    common_hour <- df %>% count(hour) %>% top_n(1, n) %>% pull(hour)
    common_day <- df %>% count(day_of_week) %>% top_n(1, n) %>% pull(day_of_week)
    common_neigh <- df %>% count(neighborhood) %>% top_n(1, n) %>% pull(neighborhood)
    
    cat("Total Incidents:", total, "\n")
    cat("Most Frequent Hour of Theft:", common_hour, "\n")
    cat("Most Frequent Day:", common_day, "\n")
    cat("Most Affected Neighborhood:", common_neigh, "\n")
  })
  
  # Plot: thefts by hour
  output$hourPlot <- renderPlot({
    df <- filtered_data()
    ggplot(df, aes(x = hour)) +
      geom_bar(fill = "steelblue") +
      labs(title = "Vehicle Thefts by Hour", x = "Hour of Day", y = "Number of Incidents") +
      theme_minimal()
  })
  
  # Plot: thefts by day
  output$dayPlot <- renderPlot({
    df <- filtered_data()
    ggplot(df, aes(x = day_of_week)) +
      geom_bar(fill = "tomato") +
      labs(title = "Vehicle Thefts by Day of Week", x = "Day", y = "Number of Incidents") +
      theme_minimal()
  })
}

# Run the app
shinyApp(ui = ui, server = server)