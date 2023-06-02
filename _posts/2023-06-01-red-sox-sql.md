---
layout: post
title: "Data Extraction Using SQL"
subtitle: "From Basics to Window Functions"
background: '/img/posts/pitcher-analysis/pitcher-analysis-BG.jpg'
---


Data Extraction Using SQL
================
Drew Colbert
2023-06-01

# Showing Off In SQL

The goal of this post is to demonstrate some of my SQL abilities and to
show that I can gather, transform, and manipulate the data how I need it
in order to perform a certain task, whether it be make a chart or make a
table to summarize the data, doing it in SQL first is the way it will be
done.

A quick outline of the data. This is statcast data collected for the
Boston Red Sox hitters and pitchers. For this post, I am only going to
be focusing on the hitters. I more in depth and interactive application
is in the works. This data is collected through ‘baseballr’ and is
uploaded to a local instance of MySQL that I created. I update this data
everyday so it is as current as 5/31/2023. All of the column definitions
can be found on the [statcast
website.](https://baseballsavant.mlb.com/csv-docs)

## The Basics

Starting with the very beginning, here we use a simple select statement
to grab some data and then visualize that data in R. The ‘WHERE’ clause
is a bit unique because I didn’t want to hard-code a value in there,
instead whenever you run this code it grabs the most current data.

The chart shows a basic density plot of the locations of pitches that
Red Sox hitters saw in their most recent game, along with each at bat
result and the location of the pitch where that event took place.

``` sql

SELECT 
  plate_x, plate_z, at_bat_event

FROM
  statcast_batters
  
WHERE
  game_date = (SELECT DATE(MAX(game_date)) FROM statcast_batters) AND at_bat_event IS NOT NULL; 
```

``` r
num_events <- length(unique(current_data$at_bat_event))
img <- readJPEG(".../catchers_view3.jpeg")
strike_zone <- data.frame(x1 = -0.705, x2 = 0.705, y1 = 1.5, y2 = 3.6)

ggplot(current_data) +
  
  background_image(img) +
  
  geom_density2d_filled(aes(x = plate_x, y = plate_z), 
                        bins = 6, 
                        show.legend = FALSE) +
  
  scale_fill_manual(values = c(alpha("#FFFFFF", 0), 
                               alpha("#ffc100", 0.7),
                               alpha("#ff9a00", 0.7),
                               alpha("#ff7400", 0.7),
                               alpha("#ff4d00", 0.7),
                               alpha("#ff0000", 0.7))) +
  
  geom_rect(data = strike_zone, aes(xmin = x1, xmax = x2, ymin = y1, ymax = y2), 
            inherit.aes = FALSE, fill = NA, color = "black", linewidth = 1.5) +
  
  geom_point(aes(x = plate_x, y = plate_z, color = at_bat_event), 
             na.rm = TRUE, 
             size = 5) +
  
  scale_color_manual(values = viridis(num_events)) +
  
  labs(color = "Result of At Bat") +
  xlim(c(-2.85,3)) +
  ylim(c(0, 5)) +
  theme_minimal() +
  theme(panel.grid = element_blank(),
        axis.title = element_blank(),
        axis.ticks = element_blank(),
        axis.text = element_blank())
```

<img src="\img\posts\red-sox-sql\pitch-density.png" style="display: block; margin: auto;" />

## Aggregate Functions

One big problem with our data is that a lot of different stats are not
directly included in our data. But that doesn’t mean that we cannot
figute out what those stats are. The most common way to avaluate a
hitter in a quick snapshot is to look at thie slash line. A slash line
consist of a hitters batting average, on-base percentage, and slugging
percentage; none of which are found in our data as it currently stands.
Luckily for us, we can use SQL to generate a slash line for each player
on our roster. The query itself looks long and complicated but it
carefully crafted to generate a slash line, which is then displayed in
the table below.

``` sql

SELECT 
  player_full_name, 
  
  ROUND(COUNT(CASE WHEN at_bat_event IN ('single', 'double', 'triple', 'home_run') THEN at_bat_event END) / COUNT(CASE WHEN at_bat_event NOT IN ('walk', 'hit_by_pitch', 'sac_fly', 'sac_bunt',
  'sac_fly_double_play', 'catcher_interf') THEN at_bat_event END), 3) AS BA,
  
  ROUND(COUNT(CASE WHEN at_bat_event IN ('single', 'double', 'triple', 'home_run', 'walk', 'hit_by_pitch') THEN at_bat_event END) / COUNT(CASE WHEN at_bat_event NOT IN ('walk', 'hit_by_pitch', 'sac_fly',
  'sac_bunt', 'sac_fly_double_play', 'catcher_interf') THEN at_bat_event END), 3) AS OBP,
  
  ROUND(((COUNT(CASE WHEN at_bat_event LIKE 'single%' THEN at_bat_event END)) + (COUNT(CASE WHEN at_bat_event LIKE 'double%' THEN at_bat_event END) * 2) + (COUNT(CASE WHEN at_bat_event LIKE 'triple%' THEN
  at_bat_event END) * 3) + (COUNT(CASE WHEN at_bat_event LIKE 'home_run%' THEN at_bat_event END) * 4)) /  COUNT(CASE WHEN at_bat_event NOT IN ('walk', 'hit_by_pitch', 'sac_fly', 'sac_bunt',
  'sac_fly_double_play', 'catcher_interf') THEN at_bat_event END), 3) AS SLG
  
FROM 
  statcast_batters
  
JOIN 
  players ON players.player_mlbam_id = statcast_batters.batter_mlbam_id
  
WHERE 
  at_bat_event is NOT NULL
  
GROUP BY 
  player_full_name
  
ORDER BY 
  BA DESC;
```

``` r
slashlines <- datatable(slash_lines,
                  rownames = FALSE,
                  colnames = c("Player Name", "BA", "OBP", "SLG"),
                  options = list(
                    pageLength = 20,
                    dom = "t",
                    ordering = FALSE,
                    searching = FALSE,
                    paging = FALSE,
                    scrollCollapse = FALSE,
                    autowidth = TRUE,
                    columnDefs = list(list(width = '50px', targets = c(0,1,2,3))),
                    initComplete = JS(
                          "function(settings, json) {",
                          "$(this.api().table().header()).css({'background-color': '#2c2c54', 'color': '#ff4d4d'});",
                          "}"
                    )
                  )
                ) %>%
          formatRound(columns = c("BA", "OBP", "SLG"), digits = 3)

```
<iframe src = '\img\posts\red-sox-sql\slashlines.html' height = '750px' width = '100%'></iframe>

## Window Functions and SubQueries

Finally, I wanted to show a players batting average over time and how it
changes throughout the year. As we know from above, we don’t have
batting average included in our data and we need a pretty long and
complicated aggregate function in order to get it, how could we find the
moving average of this number throughout the year for all of our
players? Well, one thing we can do is use a temporary table and we can
draw from that and use our window function on the temporary table to get
our required result. So in this case, the from clause is another select
statement that generates a temporary table. Our window function then
goes through this temporary table and generates a moving average for a
players batting average throughout the entire year, using all of the
previous averages. The final result is a nice dataset containing our
player names, the game dates, and the players season batting average on
those dates. We can graph this nicely using ggplot and plotly, double
clicking on a players name allows you to isolate their batting average
and you can click on more times to easily compare.

``` sql

SELECT 
    player_full_name, 
    game_date,
    ROUND(AVG(BA) over(PARTITION BY player_full_name ORDER BY game_date), 3) AS current_ba
FROM 
    (SELECT player_full_name, game_date,
    ROUND(COUNT(CASE WHEN at_bat_event IN ('single', 'double', 'triple', 'home_run') THEN at_bat_event END) / COUNT(CASE WHEN at_bat_event NOT IN ('walk', 'hit_by_pitch', 'sac_fly',
    'sac_bunt', 'sac_fly_double_play', 'catcher_interf') then at_bat_event end), 3) as BA
    FROM statcast_batters
    JOIN players ON players.player_mlbam_id = statcast_batters.batter_mlbam_id
    WHERE at_bat_event is NOT NULL
    GROUP BY player_full_name, game_date) AS raw_averages
GROUP BY 
    player_full_name, 
    game_date
ORDER BY 
    game_date;
    
```

``` r
BA_over_time$game_date <- as.Date(BA_over_time$game_date)

p <- ggplot(BA_over_time) +
  geom_line(aes(x = game_date, 
                y = current_ba, 
                group = player_full_name, 
                color = player_full_name,
                text = paste("Date: ", game_date, "<br>", "BA: ", current_ba, "<br>", "Player Name: ", player_full_name, sep = ""))) +
  scale_x_date(date_labels = "%b", date_breaks = "1 month") +
  labs(title = "Batting Averages Throughout the Season",
       x = "",
       y = "Batting Average",
       color = "Player Name") +
  theme_minimal()


averages_over_time <- ggplotly(p, tooltip = "text", hovermode = "x") %>%
  config(displayModeBar = FALSE)


```
<iframe src = '\img\posts\red-sox-sql\BA-over-time.html' height = '500px' width = '900px'></iframe>

