---
author: Nick Brazeau
title: Simulating Deterministic SIR Models in a Closed and Open Population Playground
date: 2020-04-27
tags:
  - simulation
categories:
  - R
---

Below, you will find code for creating a shiny flex-dashboard for simple SIR open and closed population models.   

```
  ---
  title: "Simulating SIR Models in a Closed and Open Population"
  date: "`r format(Sys.time(), '%B %d, %Y')`"
  editor_options:
    chunk_output_type: console
  output: flexdashboard::flex_dashboard
  runtime: shiny
  ---


  {r setup, include=FALSE}
  knitr::opts_chunk$set(echo = F, warning = F, message = F, fig.align = 'center', results = 'hide', fig.keep = 'all')
  library(tidyverse)
  library(cowplot)
  # https://rmarkdown.rstudio.com/flexdashboard/layouts.html#multiple_pages




  {r}
  #..............................................................
  # Open Model
  #..............................................................
  # stochastic simulator for the unobserved state process --> how cases are generated
  open_sir_step <- function(S, I, R, N, Beta, mu_IR, mu_L, delta.t){
    SI_events <- rbinom(n = 1, size = S, prob = 1 - exp((-(mu_L+Beta*(I/N)))*delta.t))
    SI_deaths <- rbinom(n = 1, size = SI_events, prob = mu_L/(mu_L + Beta*(I/N)))
    infected <- SI_events - SI_deaths
    IR_events <- rbinom(n = 1, size = I, prob = 1 - exp(-(mu_L+mu_IR)*delta.t))
    IR_deaths <- rbinom(n = 1, size = IR_events, prob = mu_L/(mu_L + mu_IR))
    recovered <- IR_events - IR_deaths
    recovered_deaths <- rbinom(n = 1, size = R, prob = 1 - exp(-mu_L))

    S <- S - infected + IR_deaths + recovered_deaths
    I <- I + infected - recovered - IR_deaths
    R <- R + recovered - recovered_deaths
    N <- S + I + R
    return(c("N" = N, "S" = S, "I" = I, "R" = R))
  }
  # recursively just step through time now
  open_sir_wrapper <-  function(N, I0, R0, Beta, mu_IR, mu_L, time, ...){
    # set up
    S0 <- N - I0  - R0
    time.steps <- (time - dplyr::lag(time))
    # init
    ret <- matrix(NA, nrow = length(time.steps) , ncol = 4)
    ret[1,] <- open_sir_step(S = S0,
                              I = I0, R = R0, N = N,
                              Beta = Beta,
                              mu_IR = mu_IR,
                              mu_L = mu_L,
                              delta.t = 0)

    for (t in 2:length(time.steps)) {
      ret[t,] <-  open_sir_step(S = ret[t-1, 2],
                                 I = ret[t-1, 3],
                                 R = ret[t-1, 4],
                                 N = ret[t-1, 1],
                                 Beta = Beta,
                                 mu_IR = mu_IR,
                                 mu_L = mu_L,
                                 delta.t = time.steps[t]
      )
    }
    # out
    ret <- cbind.data.frame(time, ret)
    colnames(ret) <- c("time", "N", "S", "I", "R")
    return(ret)

  }

  #..............................................................
  # Closed Model
  #..............................................................
  closed_sir_step <- function(S,I, R, N, Beta, mu_IR, delta.t){
    dN_SI <- rbinom(n = 1, size = S, prob = 1 - exp(-Beta*(I/N)*delta.t))
    dN_IR <- rbinom(n = 1, size = I, prob = 1 - exp(-mu_IR*delta.t))
    S <- S - dN_SI
    I <- I + dN_SI - dN_IR
    R <- R + dN_IR
    return(c("S" = S, "I" = I, "R" = R))
  }

  # recursively just step through time now
  closed_sir_wrapper <-  function(N, I0, R0, Beta, mu_IR, time, ...){
    # set up
    S0 <- N - I0 - R0
    time.steps <- (time - dplyr::lag(time))
    # init
    ret <- matrix(NA, nrow = length(time.steps) , ncol = 3)
    ret[1,] <- closed_sir_step(S = S0,
                                I = I0, R = R0, N = N,
                                Beta = Beta,
                                mu_IR = mu_IR, delta.t = 0)

    for (t in 2:length(time.steps)) {
      ret[t,] <-  closed_sir_step(S = ret[t-1, 1],
                                   I = ret[t-1, 2],
                                   R = ret[t-1, 3],
                                   N = N, # closed population
                                   Beta = Beta,
                                   mu_IR = mu_IR,
                                   delta.t = time.steps[t]
      )
    }
    # out
    ret <- cbind.data.frame(time, ret)
    colnames(ret) <- c("time", "S", "I", "R")
    return(ret)

  }




  ### Sidebar {.sidebar}
  {r, results='asis'}
  #..............................................................
  # Sliding Panel for Different Parameters we are estimating
  #..............................................................

  inputPanel(
    sliderInput("Time", label = "Time Observed Since Start of Epidemic",
                min = 0, max = 1e3, value = 500, step = 100),

    sliderInput("mu_L", label = "Birth & Death Rate",
                min = 0.01, max = 0.1, value = 0.01, step = 0.005),

    sliderInput("beta", label = "Beta (Effective Contact Rate)",
                min = 0, max = 1, value = 0.2, step = 0.001),

    sliderInput("mu_IR", label = "Gamma (Rate of Recovery)",
                min = 0, max = 0.5, value = 0.1, step = 0.005),

    actionLink("button", "Ruh Roh!", icon("bug"),
               style="color: #fff; background-color: #337ab7; border-color: #2e6da4")

  )





  ### SIR Closed vs. Open

  {r, results='asis', fig.align='center', fig.height=11, fig.width=8}

  renderPlot({ withProgress(message = 'Making plot', {
    # re-render button
    input$button
    #..............................................................
    # plot closed
    #..............................................................
    closed_truth <- closed_sir_wrapper(N = 1e5,
                                        I0 = 1,
                                        R0 = 0,
                                        Beta = input$beta,
                                        mu_IR = input$mu_IR,
                                        time = 1:input$Time)

    closed_sir_plot <- closed_truth %>%
      tidyr::gather(., key = "compartment", "count", 2:ncol(.)) %>%
      dplyr::mutate(Compartment = factor(compartment, levels = c("S", "I", "R"))) %>%
      ggplot() +
      geom_line(aes(x=time, y=count, color = Compartment), size = 2, alpha = 0.9) +
      scale_color_manual("", values = c("#2171b5", "#e41a1c", "#6a51a3")) +
      xlab("Time") + ylab("N") + ggtitle("SIR Closed Population") +
      scale_y_continuous(label = scales::unit_format(big.mark = ",", scale = 1, unit = "")) +
      theme_bw() +
      theme(
        plot.title = element_text(family = "Helvetica", face = "bold", vjust = 0.5,  hjust = 0.5, size = 18),
        axis.text.x = element_text(family = "Helvetica", face = "bold", hjust = 0.5, vjust = 0.5, size = 14),
        axis.text.y = element_text(family = "Helvetica", hjust = 0.5, vjust = 0.5, size = 14, angle = 45),
        legend.title = element_text(family = "Helvetica", face = "bold", vjust = 0.5, size = 15),
        legend.text = element_text(family = "Helvetica", hjust = 0.5, vjust = 0.5, size = 14),
        panel.background = element_rect(fill = "transparent"),
        plot.background = element_rect(fill = "transparent"),
        legend.background = element_rect(fill = "transparent"))



    #..............................................................
    # plot OPEN MODEL
    #..............................................................
    open_truth <- open_sir_wrapper(N = 1e5,
                                    I0 = 1,
                                    R0 = 0,
                                    Beta = input$beta,
                                    mu_IR = input$mu_IR,
                                    mu_L = input$mu_L,
                                    time = 1:input$Time)



    open_truthPlot <- open_truth %>%
      dplyr::select(-c("N")) %>%
      tidyr::gather(., key = "compartment", "count", 2:ncol(.)) %>%
      dplyr::mutate(Compartment = factor(compartment, levels = c("S", "I", "R"))) %>%
      ggplot() +
      geom_line(aes(x=time, y=count, color = Compartment), size = 2, alpha = 0.9) +
      scale_color_manual("", values = c("#2171b5", "#e41a1c", "#6a51a3")) +
      xlab("Time") + ylab("N") + ggtitle("SIR Open Population") +
      scale_y_continuous(label = scales::unit_format(big.mark = ",", scale = 1, unit = "")) +
      theme_bw() +
      theme(
        plot.title = element_text(family = "Helvetica", face = "bold", vjust = 0.5,  hjust = 0.5, size = 18),
        axis.title = element_text(family = "Helvetica", face = "bold", vjust = 0.5, size = 15),
        axis.text.x = element_text(family = "Helvetica", face = "bold", hjust = 0.5, vjust = 0.5, size = 14),
        axis.text.y = element_text(family = "Helvetica", hjust = 0.5, vjust = 0.5, size = 14, angle = 45),
        legend.title = element_text(family = "Helvetica", face = "bold", vjust = 0.5, size = 15),
        legend.text = element_text(family = "Helvetica", hjust = 0.5, vjust = 0.5, size = 14),
        panel.background = element_rect(fill = "transparent"),
        plot.background = element_rect(fill = "transparent"),
        legend.background = element_rect(fill = "transparent"))

  }) # end of progress bar

    #..............................................................
    # plot out
    #..............................................................
    toprow <- cowplot::plot_grid(closed_sir_plot, open_truthPlot, nrow = 1)


    # R0
    R0 <- round(input$beta / (input$mu_IR + input$mu_L), 2)
    bottomrow <- ggplot() + geom_text(aes(0,0,
                                          label= paste0("R[0] %~~% rho*c*D %~~% beta / (gamma + mu[L]) %~~%", R0)), parse=TRUE, size = 10, fontface = "bold") +
      theme_minimal() +
      theme(
        panel.grid = element_blank(),
        axis.text = element_blank(),
        axis.title = element_blank()
      ) + xlim(-0.1, 0.1) + ylim(-0.1, 0.1)


    cowplot::plot_grid(toprow, bottomrow, nrow = 2, rel_heights = c(4, 1))
  })
```
